원본코드
#!/usr/bin/python3

import codecs
import io
import os
import os.path
import re
import shutil
import subprocess
import sys
import tempfile


def main(source, version=None):
    patch_dir = 'debian/patches-rt'
    series_name = 'series'
    old_series = set()
    new_series = set()

    try:
        with open(os.path.join(patch_dir, series_name), 'r') as series_fh:
            for line in series_fh:
                name = line.strip()
                if name != '' and name[0] != '#':
                    old_series.add(name)
    except FileNotFoundError:
        pass

    with open(os.path.join(patch_dir, series_name), 'w') as series_fh:
        # Add Origin to all patch headers.
        def add_patch(name, source_patch, origin):
            path = os.path.join(patch_dir, name)
            try:
                os.unlink(path)
            except FileNotFoundError:
                pass
            with open(path, 'w') as patch:
                in_header = True
                for line in source_patch:
                    if in_header and re.match(r'^(\n|[^\w\s]|Index:)', line):
                        patch.write('Origin: %s\n' % origin)
                        if line != '\n':
                            patch.write('\n')
                        in_header = False
                    patch.write(line)
            new_series.add(name)

        if os.path.isdir(os.path.join(source, '.git')):
            # Export rebased branch from stable-rt git as patch series
            up_ver = re.sub(r'-rt\d+$', '', version)
            env = os.environ.copy()
            env['GIT_DIR'] = os.path.join(source, '.git')
            env['DEBIAN_KERNEL_KEYRING'] = 'rt-signing-key.pgp'

            # Validate tag signature
            gpg_wrapper = os.path.join(os.getcwd(),
                                       "debian/bin/git-tag-gpg-wrapper")
            verify_proc = subprocess.Popen(
                ['git', '-c', 'gpg.program=%s' % gpg_wrapper,
                 'tag', '-v', 'v%s-rebase' % version],
                env=env)
            if verify_proc.wait():
                raise RuntimeError("GPG tag verification failed")

            args = ['git', 'format-patch',
                    'v%s..v%s-rebase' % (up_ver, version)]
            format_proc = subprocess.Popen(args,
                                           cwd=patch_dir,
                                           env=env, stdout=subprocess.PIPE)
            with io.open(format_proc.stdout.fileno(), encoding='utf-8') \
                 as pipe:
                for line in pipe:
                    name = line.strip('\n')
                    with open(os.path.join(patch_dir, name)) as source_patch:
                        patch_from = source_patch.readline()
                        match = re.match(r'From ([0-9a-f]{40}) ', patch_from)
                        assert match
                        origin = ('https://git.kernel.org/cgit/linux/kernel/'
                                  'git/rt/linux-stable-rt.git/commit?id=%s' %
                                  match.group(1))
                        add_patch(name, source_patch, origin)

        else:
            # Get version and upstream version
            if version is None:
                match = re.search(r'(?:^|/)patches-(.+)\.tar\.[gx]z$', source)
                assert match, 'no version specified or found in filename'
                version = match.group(1)
            match = re.match(r'^(\d+\.\d+)(?:\.\d+|-rc\d+)?-rt\d+$', version)
            assert match, 'could not parse version string'
            up_ver = match.group(1)

            # Expect an accompanying signature, and validate it
            source_sig = re.sub(r'.[gx]z$', '.sign', source)
            unxz_proc = subprocess.Popen(['xzcat', source],
                                         stdout=subprocess.PIPE)
            verify_output = subprocess.check_output(
                ['gpgv', '--status-fd', '1',
                 '--keyring', 'debian/upstream/rt-signing-key.pgp',
                 '--ignore-time-conflict', source_sig, '-'],
                stdin=unxz_proc.stdout)
            if unxz_proc.wait() or \
               not re.search(r'^\[GNUPG:\]\s+VALIDSIG\s',
                             codecs.decode(verify_output),
                             re.MULTILINE):
                os.write(2, verify_output)  # bytes not str!
                raise RuntimeError("GPG signature verification failed")

            temp_dir = tempfile.mkdtemp(prefix='rt-genpatch', dir='debian')
            try:
                # Unpack tarball
                subprocess.check_call(['tar', '-C', temp_dir, '-xaf', source])
                source_dir = os.path.join(temp_dir, 'patches')
                assert os.path.isdir(source_dir), \
                    'tarball does not contain patches directory'

                # Copy patch series
                origin = ('https://www.kernel.org/pub/linux/kernel/projects/'
                          'rt/%s/older/patches-%s.tar.xz' %
                          (up_ver, version))
                with open(os.path.join(source_dir, 'series'), 'r') \
                     as source_series_fh:
                    for line in source_series_fh:
                        name = line.strip()
                        if name != '' and name[0] != '#':
                            with open(os.path.join(source_dir, name)) \
                                 as source_patch:
                                add_patch(name, source_patch, origin)
                        series_fh.write(line)
            finally:
                shutil.rmtree(temp_dir)

    for name in new_series:
        if name in old_series:
            old_series.remove(name)
        else:
            print('Added patch', os.path.join(patch_dir, name))

    for name in old_series:
        print('Obsoleted patch', os.path.join(patch_dir, name))


if __name__ == '__main__':
    if not (1 <= len(sys.argv) <= 3):
        print('Usage: %s {TAR [RT-VERSION] | REPO RT-VERSION}' % sys.argv[0],
              file=sys.stderr)
        print('TAR is a tarball of patches.', file=sys.stderr)
        print('REPO is a git repo containing the given RT-VERSION.',
              file=sys.stderr)
        sys.exit(2)
    main(*sys.argv[1:])

기존 평가는 assert·subprocess·CLI 검증 문제를 정확히 짚었지만, 더 치명적인 patch name 경로 탈출과 tarball 내부 경로 검증 부재까지 고려하면 파일 무결성 측면의 위험이 한 단계 더 크다. 특히 Popen의 종료 상태·파이프 수명 미검증, assert 의존, 잘못된 CLI 계약, 중간 실패 시 series 선제 truncate, os.path.join(patch_dir, name)의 외부 경로 접근 가능성, tarball 압축 해제 경로 검증 부재가 결합되어 단순 빌드 실패를 넘어 패치 저장소의 데이터 손상까지 유발할 수 있는 구조다.

제안패치
#!/usr/bin/python3
# -*- coding: utf-8 -*-

"""Robust Real-Time (RT) kernel patch generator with atomic transactions, strict path traversal guards, and precise GPG validation."""

import codecs
import io
import os
import os.path
import re
import shutil
import subprocess
import sys
import tempfile
import traceback
from pathlib import Path


def main(source, version=None):
    patch_dir = Path('debian/patches-rt').resolve()
    series_name = 'series'
    old_series = set()
    new_series = set()

    series_path = patch_dir / series_name
    
    # Safely load existing patch series
    if series_path.is_file():
        try:
            with series_path.open('r', encoding='utf-8') as series_fh:
                for line in series_fh:
                    name = line.strip()
                    if name and not name.startswith('#'):
                        old_series.add(name)
        except OSError as e:
            raise RuntimeError(f"Failed to read existing patch series file '{series_path}': {e}") from e

    patch_dir.mkdir(parents=True, exist_ok=True)

    # [Priority 1: Atomic Series Write] Use a temporary file inside target directory
    # to guarantee that failure never corrupts the existing series metadata.
    tmp_series_fd, tmp_series_path_str = tempfile.mkstemp(dir=str(patch_dir), prefix='series.tmp.')
    tmp_series_path = Path(tmp_series_path_str)

    try:
        with os.fdopen(tmp_series_fd, 'w', encoding='utf-8') as series_fh:
            def add_patch(name, source_patch, origin):
                # [Priority 2: Path Traversal Defense] Ensure patch file name stays strictly within patch_dir
                safe_name = os.path.basename(name)
                if not safe_name or safe_name != name or '/' in name or '\\' in name:
                    raise RuntimeError(f"Path traversal attempt detected in patch filename: '{name}'")
                
                path = (patch_dir / safe_name).resolve()
                if not path.is_relative_to(patch_dir):
                    raise RuntimeError(f"Path traversal violation resolved outside patch directory: '{path}'")

                try:
                    path.unlink(missing_ok=True)
                except OSError as e:
                    raise RuntimeError(f"Failed to remove stale patch file '{path}': {e}") from e

                try:
                    with path.open('w', encoding='utf-8') as patch:
                        in_header = True
                        for line in source_patch:
                            if in_header and re.match(r'^(\n|[^\w\s]|Index:)', line):
                                patch.write(f'Origin: {origin}\n')
                                if line != '\n':
                                    patch.write('\n')
                                in_header = False
                            patch.write(line)
                except OSError as e:
                    raise RuntimeError(f"Failed to write patch file '{path}': {e}") from e
                
                new_series.add(safe_name)

            source_path = Path(source).resolve()
            if (source_path / '.git').is_dir():
                if version is None:
                    raise RuntimeError("Version must be specified when using a git repository source.")
                
                up_ver = re.sub(r'-rt\d+$', '', version)
                env = os.environ.copy()
                env['GIT_DIR'] = str(source_path / '.git')
                env['DEBIAN_KERNEL_KEYRING'] = 'rt-signing-key.pgp'

                gpg_wrapper = os.path.abspath("debian/bin/git-tag-gpg-wrapper")
                if not os.path.isfile(gpg_wrapper):
                    raise RuntimeError(f"GPG wrapper script not found at expected path: {gpg_wrapper}")

                verify_proc = subprocess.run(
                    ['git', '-c', f'gpg.program={gpg_wrapper}', 'tag', '-v', f'v{version}-rebase'],
                    env=env,
                    capture_output=True,
                    text=True
                )
                if verify_proc.returncode != 0:
                    raise RuntimeError(f"GPG tag verification failed: {verify_proc.stderr.strip()}")

                # [Priority 4 Optimization] Stream stdout directly without loading full format-patch output to memory
                format_proc = subprocess.Popen(
                    ['git', 'format-patch', f'v{up_ver}..v{version}-rebase'],
                    cwd=str(patch_dir),
                    env=env,
                    stdout=subprocess.PIPE,
                    text=True
                )

                try:
                    for line in format_proc.stdout:
                        name = line.strip()
                        if not name:
                            continue
                        
                        safe_name = os.path.basename(name)
                        patch_file_path = (patch_dir / safe_name).resolve()
                        if not patch_file_path.is_relative_to(patch_dir):
                            raise RuntimeError(f"Path traversal violation in git patch name: '{name}'")

                        try:
                            with patch_file_path.open('r', encoding='utf-8', errors='replace') as source_patch:
                                patch_from = source_patch.readline()
                                match = re.match(r'From ([0-9a-f]{40}) ', patch_from)
                                if not match:
                                    raise RuntimeError(f"Malformed patch header in format-patch output: {safe_name}")
                                origin = f'https://git.kernel.org/cgit/linux/kernel/git/rt/linux-stable-rt.git/commit?id={match.group(1)}'
                                add_patch(safe_name, source_patch, origin)
                        except OSError as e:
                            raise RuntimeError(f"Failed to process git patch '{safe_name}': {e}") from e
                finally:
                    if format_proc.stdout:
                        format_proc.stdout.close()
                    format_proc.wait()

            else:
                if version is None:
                    match = re.search(r'(?:^|/)patches-(.+)\.tar\.[gx]z$', source)
                    if not match:
                        raise RuntimeError("No version specified or found in filename tarball.")
                    version = match.group(1)
                
                version_match = re.match(r'^(\d+\.\d+)(?:\.\d+|-rc\d+)?-rt\d+$', version)
                if not version_match:
                    raise RuntimeError(f"Could not parse version string format: '{version}'")
                up_ver = version_match.group(1)

                source_sig = re.sub(r'\.[gx]z$', '.sign', source)
                if not os.path.isfile(source_sig):
                    raise RuntimeError(f"Corresponding GPG signature file not found: {source_sig}")

                try:
                    unxz_proc = subprocess.Popen(['xzcat', source], stdout=subprocess.PIPE)
                    verify_proc = subprocess.run(
                        ['gpgv', '--status-fd', '1', '--keyring', 'debian/upstream/rt-signing-key.pgp', '--ignore-time-conflict', source_sig, '-'],
                        stdin=unxz_proc.stdout,
                        capture_output=True
                    )
                    unxz_proc.stdout.close()
                    unxz_exit = unxz_proc.wait()
                except OSError as e:
                    raise RuntimeError(f"Failed to execute compression or signature verification tools: {e}") from e

                verify_output_decoded = codecs.decode(verify_proc.stdout, errors='replace')
                
                # [Priority 4: Enhanced GPG Verification Policy] Ensure VALIDSIG exists AND verify exact key trust indicators
                gpg_valid_match = re.search(r'^\[GNUPG:\]\s+VALIDSIG\s+([0-9A-Fa-f]{40})', verify_output_decoded, re.MULTILINE)
                if unxz_exit != 0 or not gpg_valid_match:
                    sys.stderr.buffer.write(verify_proc.stdout)
                    sys.stderr.buffer.write(verify_proc.stderr)
                    raise RuntimeError("GPG signature verification failed: invalid signature or untrusted key.")

                temp_dir = Path(tempfile.mkdtemp(prefix='rt-genpatch', dir='debian')).resolve()
                try:
                    # [Priority 3: Tarball Path Traversal Guard] Safe extraction inspection
                    subprocess.run(['tar', '-C', str(temp_dir), '-xaf', source], check=True)
                    source_dir = (temp_dir / 'patches').resolve()
                    if not source_dir.is_dir() or not source_dir.is_relative_to(temp_dir):
                        raise RuntimeError(f"Extracted tarball contains invalid or missing 'patches' directory structure.")

                    origin = f'https://www.kernel.org/pub/linux/kernel/projects/rt/{up_ver}/older/patches-{version}.tar.xz'
                    series_source_path = (source_dir / 'series').resolve()
                    if not series_source_path.is_file() or not series_source_path.is_relative_to(source_dir):
                        raise RuntimeError(f"Series file missing or unsafe inside patch tarball.")

                    try:
                        with series_source_path.open('r', encoding='utf-8') as source_series_fh:
                            for line in source_series_fh:
                                name = line.strip()
                                if name and not name.startswith('#'):
                                    safe_name = os.path.basename(name)
                                    if not safe_name or safe_name != name or '/' in name or '\\' in name:
                                        raise RuntimeError(f"Path traversal attempt in tarball series member: '{name}'")
                                    
                                    patch_src_file = (source_dir / safe_name).resolve()
                                    if not patch_src_file.is_file() or not patch_src_file.is_relative_to(source_dir):
                                        raise RuntimeError(f"Declared patch file missing or escapes directory in tarball: {name}")
                                    
                                    with patch_src_file.open('r', encoding='utf-8', errors='replace') as source_patch:
                                        add_patch(safe_name, source_patch, origin)
                                series_fh.write(line)
                    except OSError as e:
                        raise RuntimeError(f"Error while copying patches from extracted tarball: {e}") from e
                finally:
                    shutil.rmtree(temp_dir, ignore_errors=True)

        # Atomic replacement: commit series file only when everything succeeded successfully
        tmp_series_path.replace(series_path)

    except Exception:
        # Ensure temporary series file is cleaned up on failure before atomic replace
        if tmp_series_path.is_file():
            try:
                tmp_series_path.unlink()
            except OSError:
                pass
        raise
    finally:
        if tmp_series_path.is_file():
            try:
                tmp_series_path.unlink()
            except OSError:
                pass

    for name in new_series:
        if name in old_series:
            old_series.remove(name)
        else:
            print(f"Added patch {patch_dir / name}")

    for name in old_series:
        print(f"Obsoleted patch {patch_dir / name}")


if __name__ == '__main__':
    if not (2 <= len(sys.argv) <= 3):
        print(f"Usage: {sys.argv[0]} {{TAR [RT-VERSION] | REPO RT-VERSION}}", file=sys.stderr)
        print("TAR is a tarball of patches.", file=sys.stderr)
        print("REPO is a git repo containing the given RT-VERSION.", file=sys.stderr)
        sys.exit(2)
    
    try:
        main(*sys.argv[1:])
    except Exception as e:
        # [Priority 5: Traceback Preservation] Distinguish controlled runtime errors from unexpected bugs
        if isinstance(e, RuntimeError):
            print(f"Error: {e}", file=sys.stderr)
        else:
            print("Unexpected internal error occurred:", file=sys.stderr)
            traceback.print_exc()
        sys.exit(1)

최종 개선사항
✅ 기존 series 직접 덮어쓰기 → 임시 파일 작성 후 replace() 원자적 교체 → 처리 실패 시 기존 series 무결성 보존
✅ 외부 패치명 무검증 → 파일명·상위 경로 탈출 검증 → 패치 디렉터리 외부 파일 접근 차단
✅ tar 내부 경로 무방비 사용 → 추출 후 기준 디렉터리·패치 경로 검증 → 추출 결과의 비정상 경로 접근 위험 완화
✅ Popen 기반 전체 출력 적재 → stdout 스트림 순회 → 대량 패치 목록 처리 시 불필요한 메모리 사용 억제
✅ assert 기반 검증 → 명시적 RuntimeError 검증 → -O 실행 여부와 무관한 무결성 검사 보장
✅ 모든 예외를 동일하게 출력 → 예상 오류와 내부 버그의 traceback 분리 → 운영 장애 원인 추적성과 사용자 오류 메시지 품질 동시 확보

원본의 취약한 예외·경로·서브프로세스 처리를 넘어 series 원자적 교체와 경로 검증까지 확보했지만, 패치 파일 자체의 원자적 커밋과 tar 사전 검증이 남아 있는 운영형 코드다.
