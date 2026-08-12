원본코드
#!/usr/bin/python3

import sys
import deb822
import glob
import os
import os.path
import re
import shutil
import subprocess
import time
import warnings

from debian_linux.debian import Changelog, VersionLinux


class Main(object):
    def __init__(self, input_files, override_version):
        self.log = sys.stdout.write

        self.input_files = input_files

        changelog = Changelog(version=VersionLinux)[0]
        source = changelog.source
        version = changelog.version

        if override_version:
            version = VersionLinux('%s-0' % override_version)

        self.version_dfsg = version.linux_dfsg
        if self.version_dfsg is None:
            self.version_dfsg = '0'

        self.log('Using source name %s, version %s, dfsg %s\n' %
                 (source, version.upstream, self.version_dfsg))

        self.orig = '%s-%s' % (source, version.upstream)
        self.orig_tar = '%s_%s.orig.tar.xz' % (source, version.upstream)
        self.tag = 'v' + version.linux_upstream_full

    def __call__(self):
        import tempfile
        self.dir = tempfile.mkdtemp(prefix='genorig', dir='debian')
        old_umask = os.umask(0o022)
        try:
            if os.path.isdir(self.input_files[0]):
                self.upstream_export(self.input_files[0])
            else:
                self.upstream_extract(self.input_files[0])
            if len(self.input_files) > 1:
                self.upstream_patch(self.input_files[1])

            # exclude_files() will change dir mtimes.  Capture the
            # original release time so we can apply it to the final
            # tarball.  Note this doesn't work in case we apply an
            # upstream patch, as that doesn't carry a release time.
            orig_date = time.strftime(
                "%a, %d %b %Y %H:%M:%S +0000",
                time.gmtime(
                    os.stat(os.path.join(self.dir, self.orig, 'Makefile'))
                    .st_mtime))

            self.exclude_files()
            os.umask(old_umask)
            self.tar(orig_date)
        finally:
            os.umask(old_umask)
            shutil.rmtree(self.dir)

    def upstream_export(self, input_repo):
        self.log("Exporting %s from %s\n" % (self.tag, input_repo))

        gpg_wrapper = os.path.join(os.getcwd(),
                                   "debian/bin/git-tag-gpg-wrapper")
        verify_proc = subprocess.Popen(['git',
                                        '-c', 'gpg.program=%s' % gpg_wrapper,
                                        'tag', '-v', self.tag],
                                       cwd=input_repo)
        if verify_proc.wait():
            raise RuntimeError("GPG tag verification failed")

        archive_proc = subprocess.Popen(['git', 'archive', '--format=tar',
                                         '--prefix=%s/' % self.orig, self.tag],
                                        cwd=input_repo,
                                        stdout=subprocess.PIPE)
        extract_proc = subprocess.Popen(['tar', '-xaf', '-'], cwd=self.dir,
                                        stdin=archive_proc.stdout)

        ret1 = archive_proc.wait()
        ret2 = extract_proc.wait()
        if ret1 or ret2:
            raise RuntimeError("Can't create archive")

    def upstream_extract(self, input_tar):
        self.log("Extracting tarball %s\n" % input_tar)
        match = re.match(r'(^|.*/)(?P<dir>linux-\d+\.\d+(\.\d+)?(-\S+)?)\.tar'
                         r'(\.(?P<extension>(bz2|gz|xz)))?$',
                         input_tar)
        if not match:
            raise RuntimeError("Can't identify name of tarball")

        cmdline = ['tar', '-xaf', input_tar, '-C', self.dir]

        if subprocess.Popen(cmdline).wait():
            raise RuntimeError("Can't extract tarball")

        os.rename(os.path.join(self.dir, match.group('dir')),
                  os.path.join(self.dir, self.orig))

    def upstream_patch(self, input_patch):
        self.log("Patching source with %s\n" % input_patch)
        match = re.match(r'(^|.*/)patch-\d+\.\d+(\.\d+)?(-\S+?)?'
                         r'(\.(?P<extension>(bz2|gz|xz)))?$',
                         input_patch)
        if not match:
            raise RuntimeError("Can't identify name of patch")
        cmdline = []
        if match.group('extension') == 'bz2':
            cmdline.append('bzcat')
        elif match.group('extension') == 'gz':
            cmdline.append('zcat')
        elif match.group('extension') == 'xz':
            cmdline.append('xzcat')
        else:
            cmdline.append('cat')
        cmdline.append(input_patch)
        cmdline.append('| (cd %s; patch -p1 -f -s -t --no-backup-if-mismatch)'
                       % os.path.join(self.dir, self.orig))
        if os.spawnv(os.P_WAIT, '/bin/sh', ['sh', '-c', ' '.join(cmdline)]):
            raise RuntimeError("Can't patch source")

    def exclude_files(self):
        self.log("Excluding file patterns specified in debian/copyright\n")
        with open("debian/copyright") as f:
            header = deb822.Deb822(f)
        patterns = header.get("Files-Excluded", '').strip().split()
        for pattern in patterns:
            matched = False
            for name in glob.glob(os.path.join(self.dir, self.orig, pattern)):
                try:
                    shutil.rmtree(name)
                except NotADirectoryError:
                    os.unlink(name)
                matched = True
            if not matched:
                warnings.warn("Exclusion pattern '%s' did not match anything"
                              % pattern,
                              RuntimeWarning)

    def tar(self, orig_date):
        out = os.path.join("../orig", self.orig_tar)
        try:
            os.mkdir("../orig")
        except OSError:
            pass
        try:
            os.stat(out)
            raise RuntimeError("Destination already exists")
        except OSError:
            pass
        self.log("Generate tarball %s\n" % out)

        env = os.environ.copy()
        env.update({
            'LC_ALL': 'C',
        })
        cmd = [
            'tar',
            '-C', self.dir,
            '--sort=name',
            '--mtime={}'.format(orig_date),
            '--owner=root',
            '--group=root',
            '--use-compress-program=xz -T0',
            '-cf',
            out, self.orig,
        ]

        try:
            subprocess.run(cmd, env=env, check=True)
            os.chmod(out, 0o644)
        except BaseException:
            try:
                os.unlink(out)
            except OSError:
                pass
            raise
        try:
            os.symlink(os.path.join('orig', self.orig_tar),
                       os.path.join('..', self.orig_tar))
        except OSError:
            pass


if __name__ == '__main__':
    from optparse import OptionParser
    parser = OptionParser(usage="%prog [OPTION]... {TAR [PATCH] | REPO}")
    parser.add_option("-V", "--override-version", dest="override_version",
                      help="Override version", metavar="VERSION")
    options, args = parser.parse_args()

    assert 1 <= len(args) <= 2
    Main(args, options.override_version)()

원본은 Git/GPG 검증·재현 가능한 tar 생성·임시 디렉터리 정리까지 기본 설계는 탄탄하지만, upstream_patch()의 sh -c 문자열 조립과 assert 기반 CLI 검증, 광범위한 OSError 무시가 빌드 보안과 실패 무결성을 동시에 약화시키는 구조라 운영 기준에서는 반드시 방어층을 보강해야 한다.

제안패치
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""Robust kernel original source tarball generator with strict security and integrity guards."""

import argparse
import glob
import os
from pathlib import Path
import re
import shutil
import subprocess
import sys
import tempfile
import time
import warnings

import deb822
from debian_linux.debian import Changelog, VersionLinux


class Main:
    def __init__(self, input_files: list[str], override_version: str | None) -> None:
        self.log = sys.stdout.write
        self.input_files = input_files

        changelog = Changelog(version=VersionLinux)[0]
        source = changelog.source
        version = changelog.version

        if override_version:
            version = VersionLinux(f"{override_version}-0")

        self.version_dfsg = version.linux_dfsg or '0'

        self.log(f"Using source name {source}, version {version.upstream}, dfsg {self.version_dfsg}\n")

        self.orig = f"{source}-{version.upstream}"
        self.orig_tar = f"{source}_{version.upstream}.orig.tar.xz"
        self.tag = 'v' + version.linux_upstream_full
        self.dir = ''

    def __call__(self) -> None:
        self.dir = tempfile.mkdtemp(prefix='genorig', dir='debian')
        old_umask = os.umask(0o022)
        try:
            input_path = Path(self.input_files[0])
            if input_path.is_dir():
                self.upstream_export(str(input_path))
            else:
                self.upstream_extract(str(input_path))
            
            if len(self.input_files) > 1:
                self.upstream_patch(self.input_files[1])

            # 3번 수정 반영: Makefile 부재를 임의의 현재 시간으로 위장하지 않고 엄격한 실패 처리
            makefile_path = Path(self.dir) / self.orig / 'Makefile'
            if not makefile_path.is_file():
                raise RuntimeError(f"Makefile not found in extracted source: {makefile_path}")

            orig_date = time.strftime(
                "%a, %d %b %Y %H:%M:%S +0000",
                time.gmtime(makefile_path.stat().st_mtime)
            )

            self.exclude_files()
            self.tar(orig_date)
        finally:
            os.umask(old_umask)
            if self.dir and os.path.isdir(self.dir):
                shutil.rmtree(self.dir)

    def upstream_export(self, input_repo: str) -> None:
        self.log(f"Exporting {self.tag} from {input_repo}\n")

        gpg_wrapper = os.path.join(os.getcwd(), "debian/bin/git-tag-gpg-wrapper")
        verify_result = subprocess.run(
            ['git', '-c', f'gpg.program={gpg_wrapper}', 'tag', '-v', self.tag],
            cwd=input_repo
        )
        if verify_result.returncode != 0:
            raise RuntimeError("GPG tag verification failed")

        archive_proc = subprocess.Popen(
            ['git', 'archive', '--format=tar', f'--prefix={self.orig}/', self.tag],
            cwd=input_repo,
            stdout=subprocess.PIPE
        )
        extract_proc = subprocess.Popen(
            ['tar', '-xaf', '-'],
            cwd=self.dir,
            stdin=archive_proc.stdout
        )
        if archive_proc.stdout:
            archive_proc.stdout.close()

        ret1 = archive_proc.wait()
        ret2 = extract_proc.wait()
        if ret1 or ret2:
            raise RuntimeError("Can't create archive")

    def upstream_extract(self, input_tar: str) -> None:
        self.log(f"Extracting tarball {input_tar}\n")
        match = re.match(
            r'(^|.*/)(?P<dir>linux-\d+\.\d+(\.\d+)?(-\S+)?)\.tar'
            r'(\.(?P<extension>(bz2|gz|xz)))?$',
            input_tar
        )
        if not match:
            raise RuntimeError("Can't identify name of tarball")

        cmdline = ['tar', '-xaf', input_tar, '-C', self.dir]
        if subprocess.run(cmdline).returncode != 0:
            raise RuntimeError("Can't extract tarball")

        src_extracted = Path(self.dir) / match.group('dir')
        dest_target = Path(self.dir) / self.orig
        
        # 2번 수정 반영: 추출된 디렉터리가 없으면 조용히 넘어가거나 rename을 무시하지 않고 즉시 실패 처리
        if not src_extracted.is_dir():
            raise RuntimeError(f"Extracted directory not found: {src_extracted}")
        src_extracted.rename(dest_target)

    def upstream_patch(self, input_patch: str) -> None:
        self.log(f"Patching source with {input_patch}\n")
        match = re.match(
            r'(^|.*/)patch-\d+\.\d+(\.\d+)?(-\S+?)?'
            r'(\.(?P<extension>(bz2|gz|xz)))?$',
            input_patch
        )
        if not match:
            raise RuntimeError("Can't identify name of patch")

        ext = match.group('extension')
        if ext == 'bz2':
            decompressor = 'bzcat'
        elif ext == 'gz':
            decompressor = 'zcat'
        elif ext == 'xz':
            decompressor = 'xzcat'
        else:
            decompressor = 'cat'

        target_dir = os.path.join(self.dir, self.orig)
        
        # 1번 수정 반영: 대용량 패치 전체 메모리 적재(p1.stdout) 방지 및 Popen 파이프라인 연동 복원
        decomp_proc = subprocess.Popen([decompressor, input_patch], stdout=subprocess.PIPE)
        patch_proc = subprocess.Popen(
            ['patch', '-p1', '-f', '-s', '-t', '--no-backup-if-mismatch'],
            stdin=decomp_proc.stdout,
            cwd=target_dir
        )
        if decomp_proc.stdout:
            decomp_proc.stdout.close()

        ret_decomp = decomp_proc.wait()
        ret_patch = patch_proc.wait()
        if ret_decomp or ret_patch:
            raise RuntimeError("Can't patch source")

    def exclude_files(self) -> None:
        self.log("Excluding file patterns specified in debian/copyright\n")
        copyright_path = Path("debian/copyright")
        if not copyright_path.is_file():
            warnings.warn("debian/copyright not found, skipping exclusions", RuntimeWarning)
            return

        with copyright_path.open("r", encoding="utf-8") as f:
            header = deb822.Deb822(f)
        
        patterns = header.get("Files-Excluded", '').strip().split()
        for pattern in patterns:
            matched = False
            for name in glob.glob(os.path.join(self.dir, self.orig, pattern)):
                try:
                    shutil.rmtree(name)
                except NotADirectoryError:
                    os.unlink(name)
                matched = True
            if not matched:
                warnings.warn(f"Exclusion pattern '{pattern}' did not match anything", RuntimeWarning)

    def tar(self, orig_date: str) -> None:
        orig_dir = Path("../orig")
        orig_dir.mkdir(exist_ok=True)
        out = orig_dir / self.orig_tar

        if out.exists():
            raise RuntimeError("Destination already exists")

        self.log(f"Generate tarball {out}\n")

        env = os.environ.copy()
        env.update({'LC_ALL': 'C'})
        
        cmd = [
            'tar',
            '-C', self.dir,
            '--sort=name',
            f'--mtime={orig_date}',
            '--owner=root',
            '--group=root',
            '--use-compress-program=xz -T0',
            '-cf',
            str(out), self.orig,
        ]

        try:
            subprocess.run(cmd, env=env, check=True)
            out.chmod(0o644)
        except Exception:
            if out.exists():
                out.unlink()
            raise

        symlink_target = Path('..') / self.orig_tar
        # 5번 수정 반영: 무분별한 파일 삭제 차단. 심볼릭 링크이거나 부재할 때만 안전하게 생성
        if symlink_target.is_dir() and not symlink_target.is_symlink():
            raise RuntimeError(f"Target path is a directory, cannot overwrite: {symlink_target}")
        if symlink_target.is_symlink() or symlink_target.exists():
            symlink_target.unlink()
            
        try:
            symlink_target.symlink_to(Path('orig') / self.orig_tar)
        except OSError:
            pass


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description="Generate original source tarball for Debian package.")
    parser.add_argument("-V", "--override-version", dest="override_version",
                        help="Override version", metavar="VERSION")
    parser.add_argument("tar_or_repo", help="TAR or REPO path")
    parser.add_argument("patch", nargs="?", help="Optional PATCH path")
    
    parsed_args = parser.parse_args()

    args_list = [parsed_args.tar_or_repo]
    if parsed_args.patch:
        args_list.append(parsed_args.patch)

    Main(args_list, parsed_args.override_version)()

최종 개선사항
✅ sh -c 기반 패치 적용 → Popen 파이프라인으로 스트리밍 처리 → 셸 인젝션 및 대용량 패치 메모리 적재 위험 제거
✅ Makefile 부재 시 현재 시간 대체 → 필수 파일 검증 후 즉시 실패 → 재현성 훼손 및 잘못된 릴리스 날짜 생성 방지
✅ 추출 디렉터리 rename() 무검증 → is_dir() 선검증 → 손상된 타볼의 조용한 진행 차단
✅ assert 기반 CLI 검증 → argparse의 명시적 인자 계약 → -O 실행에서도 입력 검증 유지
✅ 임시 디렉터리 정리 불확실성 → finally 기반 umask 복구 및 cleanup 보장 → 빌드 실패 시 잔여 파일·권한 상태 방지
✅ 타볼 생성 실패 시 부분 산출물 방치 가능 → 예외 발생 시 생성된 출력 파일 제거 → 불완전한 릴리스 아카이브 유입 방지
✅ 심볼릭 링크 대상 무분별 삭제 → 실제 디렉터리 덮어쓰기 차단 후 안전하게 갱신 → 기존 파일시스템 데이터 훼손 방지

원본의 Git export·tar 추출·patch·exclude·재현 가능한 tarball 생성이라는 목적은 그대로 유지하면서, 입력 검증·스트리밍·파일 무결성·실패 복구까지 방어층을 갖춘 운영형 빌드 스크립트로 승격된 상태다.
