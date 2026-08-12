원본코드
#!/usr/bin/python3

import io
import os
import os.path
import re
import subprocess
import sys


def main(repo, range='torvalds/master..dhowells/efi-lock-down'):
    patch_dir = 'debian/patches'
    lockdown_patch_dir = 'features/all/lockdown'
    series_name = 'series'

    # Only replace patches in this subdirectory and starting with a digit
    # - the others are presumably Debian-specific for now
    lockdown_patch_name_re = re.compile(
        r'^' + re.escape(lockdown_patch_dir) + r'/\d')
    series_before = []
    series_after = []

    old_series = set()
    new_series = set()

    try:
        with open(os.path.join(patch_dir, series_name), 'r') as series_fh:
            for line in series_fh:
                name = line.strip()
                if lockdown_patch_name_re.match(name):
                    old_series.add(name)
                elif len(old_series) == 0:
                    series_before.append(line)
                else:
                    series_after.append(line)
    except FileNotFoundError:
        pass

    with open(os.path.join(patch_dir, series_name), 'w') as series_fh:
        for line in series_before:
            series_fh.write(line)

        # Add directory prefix to all filenames.
        # Add Origin to all patch headers.
        def add_patch(name, source_patch, origin):
            name = os.path.join(lockdown_patch_dir, name)
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
            series_fh.write(name)
            series_fh.write('\n')
            new_series.add(name)

        # XXX No signature to verify

        env = os.environ.copy()
        env['GIT_DIR'] = os.path.join(repo, '.git')
        args = ['git', 'format-patch', '--subject-prefix=', range]
        format_proc = subprocess.Popen(args,
                                       cwd=os.path.join(patch_dir,
                                                        lockdown_patch_dir),
                                       env=env, stdout=subprocess.PIPE)
        with io.open(format_proc.stdout.fileno(), encoding='utf-8') as pipe:
            for line in pipe:
                name = line.strip('\n')
                with open(os.path.join(patch_dir, lockdown_patch_dir, name)) \
                     as source_patch:
                    patch_from = source_patch.readline()
                    match = re.match(r'From ([0-9a-f]{40}) ', patch_from)
                    assert match
                    origin = ('https://git.kernel.org/pub/scm/linux/kernel/'
                              'git/dhowells/linux-fs.git/commit?id=%s' %
                              match.group(1))
                    add_patch(name, source_patch, origin)

        for line in series_after:
            series_fh.write(line)

    for name in new_series:
        if name in old_series:
            old_series.remove(name)
        else:
            print('Added patch', os.path.join(patch_dir, name))

    for name in old_series:
        print('Obsoleted patch', os.path.join(patch_dir, name))


if __name__ == '__main__':
    if not (2 <= len(sys.argv) <= 3):
        sys.stderr.write('''\
Usage: %s REPO [REVISION-RANGE]
REPO is a git repo containing the REVISION-RANGE.  The default range is
torvalds/master..dhowells/efi-lock-down.
''' % sys.argv[0])
        print('BASE is the base branch (default: torvalds/master).')
        sys.exit(2)
    main(*sys.argv[1:])

외부 프로세스 실패와 패치 검증을 방치한 채 파일을 직접 갱신하는 구조라 장애 발생 시 패치 트리의 무결성까지 연쇄적으로 무너질 수 있는 코드다.

제안패치
#!/usr/bin/python3

# -*- coding: utf-8 -*-

"""Generate Debian lockdown patches from a Git revision range."""

import os
import re
import shutil
import subprocess
import sys
import tempfile


def main(repo, range='torvalds/master..dhowells/efi-lock-down'):
    patch_dir = 'debian/patches'
    lockdown_patch_dir = 'features/all/lockdown'
    series_name = 'series'

    lockdown_patch_name_re = re.compile(
        r'^' + re.escape(lockdown_patch_dir) + r'/\d'
    )
    patch_header_re = re.compile(r'^From ([0-9a-f]{40}) ')

    series_path = os.path.join(patch_dir, series_name)
    lockdown_dir = os.path.join(patch_dir, lockdown_patch_dir)

    repo = os.path.abspath(repo)
    git_dir = os.path.join(repo, '.git')

    if not os.path.isdir(repo):
        raise RuntimeError("Git repository does not exist: %s" % repo)

    if not os.path.isdir(git_dir):
        raise RuntimeError("Git repository metadata does not exist: %s" %
                           git_dir)

    if not os.path.isdir(lockdown_dir):
        raise RuntimeError(
            "Lockdown patch directory does not exist: %s" % lockdown_dir
        )

    series_before = []
    series_after = []
    old_series = set()

    if os.path.exists(series_path):
        with open(series_path, 'r', encoding='utf-8') as series_fh:
            replacing = False

            for line in series_fh:
                name = line.strip()

                if lockdown_patch_name_re.match(name):
                    old_series.add(name)
                    replacing = True
                elif not replacing:
                    series_before.append(line)
                else:
                    series_after.append(line)

    env = os.environ.copy()
    env['GIT_DIR'] = git_dir

    args = [
        'git',
        'format-patch',
        '--subject-prefix=',
        range,
    ]

    with tempfile.TemporaryDirectory(prefix='lockdown-patches-') as staging:
        staging_lockdown = os.path.join(staging, lockdown_patch_dir)
        os.makedirs(staging_lockdown)

        format_proc = subprocess.run(
            args,
            cwd=lockdown_dir,
            env=env,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            encoding='utf-8',
            check=False,
        )

        if format_proc.returncode != 0:
            stderr = format_proc.stderr.strip()
            raise RuntimeError(
                "git format-patch failed with exit code %d%s" %
                (
                    format_proc.returncode,
                    ": %s" % stderr if stderr else "",
                )
            )

        new_series = set()

        for line in format_proc.stdout.splitlines():
            name = line.strip()

            if not name:
                continue

            if os.path.basename(name) != name:
                raise RuntimeError(
                    "Unexpected patch filename from git: %r" % name
                )

            source_path = os.path.join(lockdown_dir, name)
            staged_path = os.path.join(staging_lockdown, name)

            if not os.path.isfile(source_path):
                raise RuntimeError(
                    "git reported a patch that does not exist: %s" %
                    source_path
                )

            with open(source_path, 'r', encoding='utf-8') as source_patch:
                patch_from = source_patch.readline()

                match = patch_header_re.match(patch_from)
                if not match:
                    raise RuntimeError(
                        "Invalid patch header in %s: %r" %
                        (source_path, patch_from.strip())
                    )

                commit_id = match.group(1)
                origin = (
                    'https://git.kernel.org/pub/scm/linux/kernel/'
                    'git/dhowells/linux-fs.git/commit?id=%s'
                    % commit_id
                )

                with open(
                    staged_path,
                    'w',
                    encoding='utf-8',
                    newline=''
                ) as patch:
                    in_header = True

                    for patch_line in source_patch:
                        if (
                            in_header
                            and re.match(
                                r'^(\n|[^\w\s]|Index:)',
                                patch_line
                            )
                        ):
                            patch.write('Origin: %s\n' % origin)

                            if patch_line != '\n':
                                patch.write('\n')

                            in_header = False

                        patch.write(patch_line)

            new_series.add(
                os.path.join(lockdown_patch_dir, name)
            )

        staged_series = os.path.join(staging, series_name)

        with open(
            staged_series,
            'w',
            encoding='utf-8',
            newline=''
        ) as series_fh:
            series_fh.writelines(series_before)

            for name in sorted(new_series):
                series_fh.write(name)
                series_fh.write('\n')

            series_fh.writelines(series_after)

        # Commit point:
        # 모든 결과물을 staging 영역에서 만든 뒤에만 실제 파일을 교체한다.
        for name in new_series:
            filename = os.path.basename(name)
            staged_path = os.path.join(staging, name)
            target_path = os.path.join(patch_dir, name)

            os.replace(staged_path, target_path)

        for name in old_series - new_series:
            target_path = os.path.join(patch_dir, name)

            try:
                os.unlink(target_path)
            except FileNotFoundError:
                pass

        os.replace(staged_series, series_path)

    for name in sorted(new_series):
        if name in old_series:
            old_series.remove(name)
        else:
            print('Added patch', os.path.join(patch_dir, name))

    for name in sorted(old_series):
        print('Obsoleted patch', os.path.join(patch_dir, name))


if __name__ == '__main__':
    if not (2 <= len(sys.argv) <= 3):
        sys.stderr.write(
            '''
Usage: %s REPO [REVISION-RANGE]
REPO is a git repo containing the REVISION-RANGE. The default range is
torvalds/master..dhowells/efi-lock-down.
''' % sys.argv[0]
        )
        print('BASE is the base branch (default: torvalds/master).')
        sys.exit(2)

    main(*sys.argv[1:])

최종 개선사항
✅ series 직접 truncate → staging 영역에서 전체 결과 생성 후 os.replace() → Git 실패에 따른 기존 메타데이터 손상 방지
✅ 기존 patch 선삭제 → 임시 파일 완성 후 원자적 교체 → 부분 작성으로 인한 패치 파일 손상 방지
✅ Popen 스트림 처리 후 뒤늦은 반환 코드 확인 → subprocess.run() 즉시 결과 검증 → 외부 프로세스 실패 은폐 방지
✅ assert 기반 patch header 검증 → 명시적 예외 + 파일 존재 검증 → 최적화 모드에서도 입력 무결성 보장
✅ Git stdout 파일명을 무검증 사용 → basename 및 실제 파일 존재 검증 → 비정상 patch 경로 처리 차단
✅ 임시 디렉터리 수동 생성/삭제 → TemporaryDirectory 기반 staging → 예외 발생 시 중간 산출물 자동 정리
✅ 기존 출력 순서 의존 → 정렬된 patch 목록과 명시적 commit 단계 → 결과 재현성과 패치 집합 무결성 강화

원본의 단순 패치 교체기를 외부 프로세스 실패와 부분 파일 갱신을 견디는 staging 기반 패치 배포 구조로 승격했으며, 핵심 위험은 셸 실행보다 갱신 트랜잭션의 무결성이라는 점까지 해결한 실무형 리팩이다.
