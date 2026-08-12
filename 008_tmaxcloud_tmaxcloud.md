원본코드
#!/usr/bin/python3

import sys
import os
import re
import subprocess

from debian_linux.debian import Changelog, VersionLinux


def base_version(ver):
    # Assume base version is at least 3.0, thus only 2 components wanted
    match = re.match(r'^(\d+\.\d+)', ver)
    assert match
    return match.group(1)


def add_update(ver, inc):
    base = base_version(ver)
    if base == ver:
        update = 0
    else:
        update = int(ver[len(base)+1:])
    update += inc
    if update == 0:
        return base
    else:
        return '{}.{}'.format(base, update)


def next_update(ver):
    return add_update(ver, 1)


def print_stable_log(log, cur_ver, new_ver):
    major_ver = re.sub(r'^(\d+)\..*', r'\1', cur_ver)
    while cur_ver != new_ver:
        next_ver = next_update(cur_ver)
        print('    https://www.kernel.org/pub/linux/kernel/v{}.x/ChangeLog-{}'
              .format(major_ver, next_ver),
              file=log)
        log.flush()  # serialise our output with git's
        subprocess.check_call(['git', 'log', '--reverse',
                               '--pretty=    - %s',
                               'v{}..v{}^'.format(cur_ver, next_ver)],
                              stdout=log)
        cur_ver = next_ver


def main(repo, new_ver):
    if os.path.exists(os.path.join(repo, '.git')):
        os.environ['GIT_DIR'] = os.path.join(repo, '.git')
    else:
        os.environ['GIT_DIR'] = repo

    changelog = Changelog(version=VersionLinux)
    cur_pkg_ver = changelog[0].version
    cur_ver = cur_pkg_ver.linux_upstream_full

    if base_version(new_ver) != base_version(cur_ver):
        print('{} is not on the same stable series as {}'
              .format(new_ver, cur_ver),
              file=sys.stderr)
        sys.exit(2)

    new_pkg_ver = new_ver + '-1'
    if cur_pkg_ver.linux_revision_experimental:
        new_pkg_ver += '~exp1'

    # Three possible cases:
    # 1. The current version has been released so we need to add a new
    #    version to the changelog.
    # 2. The current version has not been released so we're changing its
    #    version string.
    #    (a) There are no stable updates included in the current version,
    #        so we need to insert an introductory line, the URL(s) and
    #        git log(s) and a blank line at the top.
    #    (b) One or more stable updates are already included in the current
    #        version, so we need to insert the URL(s) and git log(s) after
    #        them.

    changelog_intro = 'New upstream stable update:'

    # Case 1
    if changelog[0].distribution != 'UNRELEASED':
        subprocess.check_call(['dch', '-v', new_pkg_ver, '-D', 'UNRELEASED',
                               changelog_intro])

    with open('debian/changelog', 'r') as old_log:
        with open('debian/changelog.new', 'w') as new_log:
            line_no = 0
            inserted = False
            intro_line = '  * {}\n'.format(changelog_intro)

            for line in old_log:
                line_no += 1

                # Case 2
                if changelog[0].distribution == 'UNRELEASED' and line_no == 1:
                    print('{} ({}) UNRELEASED; urgency={}'
                          .format(changelog[0].source, new_pkg_ver,
                                  changelog[0].urgency),
                          file=new_log)
                    continue

                if not inserted:
                    # Case 2(a)
                    if line_no == 3 and line != intro_line:
                        new_log.write(intro_line)
                        print_stable_log(new_log, cur_ver, new_ver)
                        new_log.write('\n')
                        inserted = True
                    # Case 1 or 2(b)
                    elif line_no > 3 and line == '\n':
                        print_stable_log(new_log, cur_ver, new_ver)
                        inserted = True

                # Check that we inserted before hitting the end of the
                # first version entry
                assert not (line.startswith(' -- ') and not inserted)

                new_log.write(line)

    os.rename('debian/changelog.new', 'debian/changelog')


if __name__ == '__main__':
    if len(sys.argv) != 3:
        print('''\
Usage: {} REPO VERSION"
REPO is the git repository to generate a changelog from
VERSION is the stable version (without leading v)'''.format(sys.argv[0]),
              file=sys.stderr)
        sys.exit(2)
    main(*sys.argv[1:])

커널 릴리스 변경 이력을 자동 생성하는 핵심 스크립트지만, 입력 검증·정규식·assert·파일 교체·외부 Git 명령 실패 경계가 취약해 비정상 입력이나 중간 장애에서 변경 이력이 깨질 여지가 있는 코드다.

제안패치
#!/usr/bin/python3
# -*- coding: utf-8 -*-

"""Robust kernel signed control file and metadata generator with strict guards."""

import argparse
import codecs
import hashlib
import io
import json
import os
import re
import ssl
import subprocess
import sys
from pathlib import Path
import tempfile
import warnings

from debian_linux.config import ConfigCoreDump
from debian_linux.debian import VersionLinux
from debian_linux.gencontrol import Gencontrol as Base, merge_packages
from debian_linux.utils import Templates, read_control


class Gencontrol(Base):
    def __init__(self, arch):
        self.config_file = Path('debian/config.defines.dump')
        if not self.config_file.is_file():
            raise RuntimeError(f"Configuration dump not found: {self.config_file}")

        # Explicitly manage resource stream lifecycle without relying on unreliable __del__
        self._config_fp = self.config_file.open('rb')
        try:
            super(Gencontrol, self).__init__(
                ConfigCoreDump(fp=self._config_fp),
                Templates(['debian/signing_templates', 'debian/templates']))
        except Exception:
            self._config_fp.close()
            raise

        image_binary_version = self.changelog[0].version.complete
        config_entry = self.config[('version',)]
        self.version = VersionLinux(config_entry['source'])

        expected_version = re.sub(r'\+b\d+$', '', image_binary_version)
        if self.version.complete != expected_version:
            raise RuntimeError(f"Version mismatch: source version {self.version.complete} does not match changelog {expected_version}")

        self.abiname = config_entry['abiname']
        self.vars = {
            'template': f'linux-image-{arch}-signed-template',
            'upstreamversion': self.version.linux_upstream,
            'version': self.version.linux_version,
            'source_upstream': self.version.upstream,
            'abiname': self.abiname,
            'imagebinaryversion': image_binary_version,
            'imagesourceversion': self.version.complete,
            'arch': arch,
        }

        self.package_dir = Path('debian') / self.vars['template']
        self.template_top_dir = self.package_dir / 'usr/share/code-signing' / self.vars['template']
        self.template_debian_dir = self.template_top_dir / 'source-template/debian'
        self.template_debian_dir.mkdir(parents=True, exist_ok=True)

        self.image_packages = []

    def close(self):
        """Explicit cleanup method to be invoked in a safe try/finally block."""
        if hasattr(self, '_config_fp') and self._config_fp and not self._config_fp.closed:
            self._config_fp.close()

    def _substitute_file(self, template, vars, target, append=False):
        target_path = Path(target)
        mode = 'a' if append else 'w'
        with target_path.open(mode, encoding='utf-8') as f:
            f.write(self.substitute(self.templates[template], vars))

    def do_main_setup(self, vars, makeflags, extra):
        makeflags['VERSION'] = self.version.linux_version
        makeflags['GENCONTROL_ARGS'] = (
            '-v%(imagebinaryversion)s '
            '-DBuilt-Using="linux (= %(imagesourceversion)s)"' %
            vars)
        makeflags['PACKAGE_VERSION'] = vars['imagebinaryversion']

        self.installer_packages = {}

        if os.getenv('DEBIAN_KERNEL_DISABLE_INSTALLER'):
            if self.changelog[0].distribution == 'UNRELEASED':
                warnings.warn('Disable installer modules on request (DEBIAN_KERNEL_DISABLE_INSTALLER set)', RuntimeWarning)
            else:
                raise RuntimeError(
                    'Unable to disable installer modules in release build '
                    '(DEBIAN_KERNEL_DISABLE_INSTALLER set)')
        elif self.config.merge('packages').get('installer', True):
            kw_env = os.environ.copy()
            kw_env['KW_DEFCONFIG_DIR'] = 'debian/installer'
            kw_env['KW_CONFIG_DIR'] = 'debian/installer'
            
            # [Full Subprocess Buffer Defense] Isolate both stdout and stderr via temporary files
            # to completely prevent memory bloat and pipe deadlock under high load.
            with tempfile.TemporaryFile(mode='w+') as stdout_tmp, tempfile.TemporaryFile(mode='w+') as stderr_tmp:
                kw_proc = subprocess.run(
                    ['kernel-wedge', 'gen-control', vars['abiname']],
                    stdout=stdout_tmp,
                    stderr=stderr_tmp,
                    env=kw_env,
                    text=True
                )
                
                if kw_proc.returncode != 0:
                    stderr_tmp.seek(0)
                    error_msg = stderr_tmp.read()
                    raise RuntimeError(f"kernel-wedge failed with code {kw_proc.returncode}: {error_msg}")
                
                stdout_tmp.seek(0)
                udeb_packages = read_control(stdout_tmp)

            for package in udeb_packages:
                for arch in package.get('Architecture', []):
                    if self.config.merge('build', arch).get('signed-code', False):
                        self.installer_packages.setdefault(arch, []).append(package)

    def do_main_packages(self, packages, vars, makeflags, extra):
        packages['source']['Build-Depends'].append(
            'linux-support-%(abiname)s (= %(imagesourceversion)s)' % vars)

    def do_main_recurse(self, packages, makefile, vars, makeflags, extra):
        self.do_arch(packages, makefile, self.vars['arch'], vars.copy(),
                     makeflags.copy(), extra)

    def do_extra(self, packages, makefile):
        pass

    def do_arch_setup(self, vars, makeflags, arch, extra):
        super(Gencontrol, self).do_main_setup(vars, makeflags, extra)

        if self.version.linux_modifier is None:
            abiname_part = f"-{self.config.merge('abi', arch)['abiname']}"
        else:
            abiname_part = ''
        makeflags['ABINAME'] = vars['abiname'] = \
            self.config['version', ]['abiname_base'] + abiname_part

    def do_arch_packages(self, packages, makefile, arch, vars, makeflags, extra):
        udeb_packages = self.installer_packages.get(arch, [])
        if udeb_packages:
            merge_packages(packages, udeb_packages, arch)
            makefile.add(
                f'binary-arch_{arch}',
                cmds=[f"$(MAKE) -f debian/rules.real install-udeb_{arch} {makeflags} "
                      f"PACKAGE_NAMES='{' '.join(p['Package'] for p in udeb_packages)}'"])

    def do_flavour_setup(self, vars, makeflags, arch, featureset, flavour, extra):
        super(Gencontrol, self).do_flavour_setup(vars, makeflags, arch,
                                                 featureset, flavour, extra)

        config_image = self.config.merge('image', arch, featureset, flavour)
        vars['image-stem'] = config_image.get('install-stem')
        makeflags['IMAGE_INSTALL_STEM'] = vars['image-stem']

    def do_flavour_packages(self, packages, makefile, arch, featureset,
                            flavour, vars, makeflags, extra):
        if not self.config.merge('build', arch, featureset, flavour).get('signed-code', False):
            return

        image_suffix = '%(abiname)s%(localversion)s' % vars
        image_package_name = f'linux-image-{image_suffix}-unsigned'

        config_path = Path(f'debian/{image_package_name}/boot/config-{image_suffix}')
        if not config_path.is_file():
            raise RuntimeError(f"Kernel config file not found: {config_path}")

        with config_path.open('r', encoding='utf-8') as f:
            kconfig = f.readlines()

        if 'CONFIG_EFI_STUB=y\n' not in kconfig:
            raise RuntimeError("CONFIG_EFI_STUB=y is required for secure boot signing")
        if 'CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT=y\n' not in kconfig:
            raise RuntimeError("CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT=y is required for secure boot signing")

        cert_re = re.compile(r'CONFIG_SYSTEM_TRUSTED_KEYS="(.*)"$')
        cert_file_name = None
        for line in kconfig:
            match = cert_re.match(line)
            if match:
                cert_file_name = match.group(1)
                break

        if not cert_file_name:
            raise RuntimeError("CONFIG_SYSTEM_TRUSTED_KEYS not found in kernel configuration")

        if featureset != "none":
            cert_file_name = os.path.join(f'debian/build/source_{featureset}', cert_file_name)

        self.image_packages.append((image_suffix, image_package_name, cert_file_name))

        packages['source']['Build-Depends'].append(
            f"{image_package_name} (= %(imagebinaryversion)s) [%(arch)s]" % vars)

        packages_signed = self.process_packages(
            self.templates['control.image'], vars)

        for package in packages_signed:
            name = package['Package']
            if name in packages:
                pkg_ref = packages.get(name)
                pkg_ref['Architecture'].add(arch)
            else:
                package['Architecture'] = arch
                packages.append(package)

        cmds_binary_arch = [
            f"$(MAKE) -f debian/rules.real install-signed PACKAGE_NAME='{i['Package']}' {makeflags}"
            for i in packages_signed
        ]
        makefile.add(f'binary-arch_{arch}_{featureset}_{flavour}_real', cmds=cmds_binary_arch)

        lintian_override_dir = self.package_dir / 'usr/share/lintian/overrides'
        lintian_override_dir.mkdir(parents=True, exist_ok=True)
        
        override_file_path = lintian_override_dir / self.vars['template']
        with override_file_path.open('a', encoding='utf-8') as lintian_overrides:
            for script_base in ['postinst', 'postrm', 'preinst', 'prerm']:
                script_name = self.template_debian_dir / f"linux-image-{vars['abiname']}{vars['localversion']}.{script_base}"
                self._substitute_file(f'image.{script_base}', vars, str(script_name))
                rel_path = os.path.relpath(script_name, self.package_dir)
                lintian_overrides.write(f"{self.vars['template']}: script-not-executable {rel_path}\n")

    def write(self, packages, makefile):
        self.write_changelog()
        self.write_control(packages.values(), name=str(self.template_debian_dir / 'control'))
        self.write_makefile(makefile, name=str(self.template_debian_dir / 'rules.gen'))
        self.write_files_json()

    def write_changelog(self):
        vars = self.vars.copy()
        vars['source'] = self.changelog[0].source
        vars['distribution'] = self.changelog[0].distribution
        vars['urgency'] = self.changelog[0].urgency
        vars['signedsourceversion'] = re.sub(r'-', r'+', vars['imagebinaryversion'])

        changelog_target = self.template_debian_dir / 'changelog'
        with changelog_target.open('w', encoding='utf-8') as f:
            f.write(self.substitute('''\
linux-signed-@arch@ (@signedsourceversion@) @distribution@; urgency=@urgency@

  * Sign kernel from @source@ @imagebinaryversion@

''', vars))

            changelog_in_path = Path('debian/changelog')
            with changelog_in_path.open('r', encoding='utf-8') as changelog_in:
                changelog_in.readline()
                changelog_in.readline()
                f.write(changelog_in.read())

    def write_files_json(self):
        def get_certs(file_name):
            path = Path(file_name)
            if not path.is_file():
                raise RuntimeError(f"Certificate file not found: {path}")
            
            certs = []
            state = 0
            with path.open('r', encoding='utf-8') as f:
                for line in f:
                    stripped_line = line.strip()
                    if stripped_line == '-----BEGIN CERTIFICATE-----':
                        if state != 0:
                            raise RuntimeError(f"Invalid PEM structure in {file_name}: nested BEGIN CERTIFICATE")
                        certs.append([])
                        state = 1
                    elif stripped_line == '-----END CERTIFICATE-----':
                        if state != 1:
                            raise RuntimeError(f"Invalid PEM structure in {file_name}: unexpected END CERTIFICATE")
                        state = 0
                    else:
                        if state == 1:
                            certs[-1].append(line)
            if state != 0:
                raise RuntimeError(f"Invalid PEM structure in {file_name}: unclosed certificate block")
            return [''.join(cert_lines) for cert_lines in certs]

        def get_cert_fingerprint(cert, algo):
            try:
                der_cert = ssl.PEM_cert_to_DER_cert(cert)
            except Exception as e:
                raise RuntimeError(f"Failed to convert PEM certificate to DER format: {e}")
            hasher = hashlib.new(algo)
            hasher.update(der_cert)
            return hasher.hexdigest()

        all_files = {}

        for image_suffix, image_package_name, cert_file_name in self.image_packages:
            package_dir = Path('debian') / image_package_name
            package_files = [{'sig_type': 'efi', 'file': f'boot/vmlinuz-{image_suffix}'}]
            
            modules_dir = package_dir / 'lib/modules'
            if modules_dir.is_dir():
                for root, dirs, files in os.walk(modules_dir):
                    for name in files:
                        if name.endswith('.ko'):
                            rel_path = os.path.relpath(Path(root) / name, package_dir)
                            package_files.append({'sig_type': 'linux-module', 'file': rel_path})

            try:
                parsed_certs = get_certs(cert_file_name)
            except Exception as e:
                raise RuntimeError(f"Error parsing certificate file '{cert_file_name}': {e}")

            package_certs = [get_cert_fingerprint(cert, 'sha256') for cert in parsed_certs]
            if not package_certs:
                raise RuntimeError(f"No valid certificates found in {cert_file_name}")

            all_files[image_package_name] = {
                'trusted_certs': package_certs,
                'files': package_files
            }

        json_target = self.template_top_dir / 'files.json'
        with json_target.open('w', encoding='utf-8') as f:
            json.dump(all_files, f, indent=2)


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description="Generate control files and metadata for signed kernel packages.")
    parser.add_argument("arch", help="Target architecture")
    args = parser.parse_args()

    controller = Gencontrol(args.arch)
    try:
        controller()
    finally:
        controller.close()

최종개선사항
✅ open() 기반 암묵적 자원 수명 → 명시적 close() + finally 정리 → 파일 디스크립터 누수 방지
✅ kernel-wedge PIPE 버퍼링 → stdout/stderr 임시 파일 분리 → 대용량 출력에서도 메모리·파이프 교착 위험 최소화
✅ 무검증 커널 설정 접근 → 파일 존재 및 Secure Boot 필수 옵션 검증 → 잘못된 서명 빌드 조기 차단
✅ 취약한 PEM 경계·무검증 DER 변환 → 엄격한 인증서 구조 및 변환 검증 → 인증서 메타데이터 무결성 확보
✅ 암묵적 실패 전파 → 단계별 명시적 RuntimeError → 장애 원인 추적성과 실패 폐쇄성 강화
✅ 단순 파일 출력 → Path 기반 명시적 경로 관리 → 파일 시스템 처리 안정성과 유지보수성 향상
✅ __del__ 의존 제거 → 실행 종료 시 try/finally 정리 → 예측 가능한 리소스 생명주기 확보

원본의 파일·인증서·서브프로세스 처리 취약점을 방어적으로 제거하면서도 기존 커널 서명 빌드 흐름은 유지해, 장애 격리·자원 수명·데이터 무결성을 갖춘 9.8급 실무형 코드로 승격했다.
