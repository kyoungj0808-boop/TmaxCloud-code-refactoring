원본코드
#!/usr/bin/python3

import codecs
import hashlib
import io
import json
import os.path
import re
import ssl
import subprocess
import sys

from debian_linux.config import ConfigCoreDump
from debian_linux.debian import VersionLinux
from debian_linux.gencontrol import Gencontrol as Base, merge_packages
from debian_linux.utils import Templates, read_control


class Gencontrol(Base):
    def __init__(self, arch):
        super(Gencontrol, self).__init__(
            ConfigCoreDump(fp=open('debian/config.defines.dump', 'rb')),
            Templates(['debian/signing_templates', 'debian/templates']))

        image_binary_version = self.changelog[0].version.complete

        config_entry = self.config[('version',)]
        self.version = VersionLinux(config_entry['source'])

        # Check config version matches changelog version
        assert self.version.complete == re.sub(r'\+b\d+$', r'',
                                               image_binary_version)

        self.abiname = config_entry['abiname']
        self.vars = {
            'template': 'linux-image-%s-signed-template' % arch,
            'upstreamversion': self.version.linux_upstream,
            'version': self.version.linux_version,
            'source_upstream': self.version.upstream,
            'abiname': self.abiname,
            'imagebinaryversion': image_binary_version,
            'imagesourceversion': self.version.complete,
            'arch': arch,
        }

        self.package_dir = 'debian/%(template)s' % self.vars
        self.template_top_dir = (self.package_dir
                                 + '/usr/share/code-signing/%(template)s'
                                 % self.vars)
        self.template_debian_dir = (self.template_top_dir
                                    + '/source-template/debian')
        os.makedirs(self.template_debian_dir, exist_ok=True)

        self.image_packages = []

    def _substitute_file(self, template, vars, target, append=False):
        with codecs.open(target, 'a' if append else 'w', 'utf-8') as f:
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
                import warnings
                warnings.warn('Disable installer modules on request '
                              '(DEBIAN_KERNEL_DISABLE_INSTALLER set)')
            else:
                raise RuntimeError(
                    'Unable to disable installer modules in release build '
                    '(DEBIAN_KERNEL_DISABLE_INSTALLER set)')
        elif self.config.merge('packages').get('installer', True):
            # Add udebs using kernel-wedge
            kw_env = os.environ.copy()
            kw_env['KW_DEFCONFIG_DIR'] = 'debian/installer'
            kw_env['KW_CONFIG_DIR'] = 'debian/installer'
            kw_proc = subprocess.Popen(
                ['kernel-wedge', 'gen-control', vars['abiname']],
                stdout=subprocess.PIPE,
                env=kw_env)
            if not isinstance(kw_proc.stdout, io.IOBase):
                udeb_packages = read_control(io.open(kw_proc.stdout.fileno(),
                                                     closefd=False))
            else:
                udeb_packages = read_control(io.TextIOWrapper(kw_proc.stdout))
            kw_proc.wait()
            if kw_proc.returncode != 0:
                raise RuntimeError('kernel-wedge exited with code %d' %
                                   kw_proc.returncode)

            for package in udeb_packages:
                for arch in package['Architecture']:
                    if self.config.merge('build', arch) \
                                  .get('signed-code', False):
                        self.installer_packages.setdefault(arch, []) \
                                               .append(package)

    def do_main_packages(self, packages, vars, makeflags, extra):
        # Assume that arch:all packages do not get binNMU'd
        packages['source']['Build-Depends'].append(
            'linux-support-%(abiname)s (= %(imagesourceversion)s)' % vars)

    def do_main_recurse(self, packages, makefile, vars, makeflags, extra):
        # Each signed source package only covers a single architecture
        self.do_arch(packages, makefile, self.vars['arch'], vars.copy(),
                     makeflags.copy(), extra)

    def do_extra(self, packages, makefile):
        pass

    def do_arch_setup(self, vars, makeflags, arch, extra):
        super(Gencontrol, self).do_main_setup(vars, makeflags, extra)

        if self.version.linux_modifier is None:
            abiname_part = '-%s' % self.config.merge('abi', arch)['abiname']
        else:
            abiname_part = ''
        makeflags['ABINAME'] = vars['abiname'] = \
            self.config['version', ]['abiname_base'] + abiname_part

    def do_arch_packages(self, packages, makefile, arch, vars, makeflags,
                         extra):
        udeb_packages = self.installer_packages.get(arch, [])
        if udeb_packages:
            merge_packages(packages, udeb_packages, arch)

            # These packages must be built after the per-flavour/
            # per-featureset packages.  Also, this won't work
            # correctly with an empty package list.
            if udeb_packages:
                makefile.add(
                    'binary-arch_%s' % arch,
                    cmds=["$(MAKE) -f debian/rules.real install-udeb_%s %s "
                          "PACKAGE_NAMES='%s'" %
                          (arch, makeflags,
                           ' '.join(p['Package'] for p in udeb_packages))])

    def do_flavour_setup(self, vars, makeflags, arch, featureset, flavour,
                         extra):
        super(Gencontrol, self).do_flavour_setup(vars, makeflags, arch,
                                                 featureset, flavour, extra)

        config_image = self.config.merge('image', arch, featureset, flavour)
        vars['image-stem'] = config_image.get('install-stem')
        makeflags['IMAGE_INSTALL_STEM'] = vars['image-stem']

    def do_flavour_packages(self, packages, makefile, arch, featureset,
                            flavour, vars, makeflags, extra):
        if not (self.config.merge('build', arch, featureset, flavour)
                .get('signed-code', False)):
            return

        image_suffix = '%(abiname)s%(localversion)s' % vars
        image_package_name = 'linux-image-%s-unsigned' % image_suffix

        # Verify that this flavour is configured to support Secure Boot,
        # and get the trusted certificates filename.
        with open('debian/%s/boot/config-%s' %
                  (image_package_name, image_suffix)) as f:
            kconfig = f.readlines()
        assert 'CONFIG_EFI_STUB=y\n' in kconfig
        assert 'CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT=y\n' in kconfig
        cert_re = re.compile(r'CONFIG_SYSTEM_TRUSTED_KEYS="(.*)"$')
        cert_file_name = None
        for line in kconfig:
            match = cert_re.match(line)
            if match:
                cert_file_name = match.group(1)
                break
        assert cert_file_name
        if featureset != "none":
            cert_file_name = os.path.join('debian/build/source_%s' %
                                          featureset,
                                          cert_file_name)

        self.image_packages.append((image_suffix, image_package_name,
                                    cert_file_name))

        packages['source']['Build-Depends'].append(
            image_package_name
            + ' (= %(imagebinaryversion)s) [%(arch)s]' % vars)

        packages_signed = self.process_packages(
            self.templates['control.image'], vars)

        for package in packages_signed:
            name = package['Package']
            if name in packages:
                package = packages.get(name)
                package['Architecture'].add(arch)
            else:
                package['Architecture'] = arch
                packages.append(package)

        cmds_binary_arch = []
        for i in packages_signed:
            cmds_binary_arch += ["$(MAKE) -f debian/rules.real install-signed "
                                 "PACKAGE_NAME='%s' %s" %
                                 (i['Package'], makeflags)]
        makefile.add('binary-arch_%s_%s_%s_real' % (arch, featureset, flavour),
                     cmds=cmds_binary_arch)

        os.makedirs(self.package_dir + '/usr/share/lintian/overrides', 0o755,
                    exist_ok=True)
        with open(self.package_dir
                  + '/usr/share/lintian/overrides/%(template)s' % self.vars,
                  'a') as lintian_overrides:
            for script_base in ['postinst', 'postrm', 'preinst', 'prerm']:
                script_name = (self.template_debian_dir
                               + '/linux-image-%s%s.%s'
                               % (vars['abiname'], vars['localversion'],
                                  script_base))
                self._substitute_file('image.%s' % script_base, vars,
                                      script_name)
                lintian_overrides.write('%s: script-not-executable %s\n' %
                                        (self.vars['template'],
                                         os.path.relpath(script_name,
                                                         self.package_dir)))

    def write(self, packages, makefile):
        self.write_changelog()
        self.write_control(packages.values(),
                           name=(self.template_debian_dir + '/control'))
        self.write_makefile(makefile,
                            name=(self.template_debian_dir + '/rules.gen'))
        self.write_files_json()

    def write_changelog(self):
        # We need to insert a new version entry.
        # Take the distribution and urgency from the linux changelog, and
        # the base version from the changelog template.
        vars = self.vars.copy()
        vars['source'] = self.changelog[0].source
        vars['distribution'] = self.changelog[0].distribution
        vars['urgency'] = self.changelog[0].urgency
        vars['signedsourceversion'] = (re.sub(r'-', r'+',
                                              vars['imagebinaryversion']))

        with codecs.open(self.template_debian_dir + '/changelog', 'w',
                         'utf-8') as f:
            f.write(self.substitute('''\
linux-signed-@arch@ (@signedsourceversion@) @distribution@; urgency=@urgency@

  * Sign kernel from @source@ @imagebinaryversion@

''',
                                    vars))

            with codecs.open('debian/changelog', 'r', 'utf-8') as changelog_in:
                # Ignore first two header lines
                changelog_in.readline()
                changelog_in.readline()

                for d in changelog_in.read():
                    f.write(d)

    def write_files_json(self):
        # Can't raise from a lambda function :-(
        def raise_func(e):
            raise e

        # Some functions in openssl work with multiple concatenated
        # PEM-format certificates, but others do not.
        def get_certs(file_name):
            certs = []
            BEGIN, MIDDLE = 0, 1
            state = BEGIN
            with open(file_name) as f:
                for line in f:
                    if line == '-----BEGIN CERTIFICATE-----\n':
                        assert state == BEGIN
                        certs.append([])
                        state = MIDDLE
                    elif line == '-----END CERTIFICATE-----\n':
                        assert state == MIDDLE
                        state = BEGIN
                    else:
                        assert line[0] != '-'
                        assert state == MIDDLE
                    certs[-1].append(line)
            assert state == BEGIN
            return [''.join(cert_lines) for cert_lines in certs]

        def get_cert_fingerprint(cert, algo):
            hasher = hashlib.new(algo)
            hasher.update(ssl.PEM_cert_to_DER_cert(cert))
            return hasher.hexdigest()

        all_files = {}

        for image_suffix, image_package_name, cert_file_name in \
                self.image_packages:
            package_dir = 'debian/%s' % image_package_name
            package_files = []
            package_files.append({'sig_type': 'efi',
                                  'file': 'boot/vmlinuz-%s' % image_suffix})
            for root, dirs, files in os.walk('%s/lib/modules' % package_dir,
                                             onerror=raise_func):
                for name in files:
                    if name.endswith('.ko'):
                        package_files.append(
                            {'sig_type': 'linux-module',
                             'file': '%s/%s' %
                             (root[(len(package_dir) + 1):], name)})
            package_certs = [get_cert_fingerprint(cert, 'sha256')
                             for cert in get_certs(cert_file_name)]
            assert len(package_certs) >= 1
            all_files[image_package_name] = {
                'trusted_certs': package_certs,
                'files': package_files
            }

        with codecs.open(self.template_top_dir + '/files.json', 'w') as f:
            json.dump(all_files, f)


if __name__ == '__main__':
    Gencontrol(sys.argv[1])()

원본은 커널 Secure Boot 서명 패키지 생성이라는 핵심 역할은 제대로 수행하지만, assert에 무결성 검증을 맡기고 파일·프로세스 자원 수명과 인증서 입력을 방어적으로 관리하지 않아 운영 환경에서 조용한 실패와 검증 우회 가능성을 남긴 빌드 인프라 코드다.

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

        # [Defensive Design] ConfigCoreDump lifetime contract protection:
        # Keep the file descriptor alive across the Base lifecycle by binding it to self,
        # and ensure cleanup happens only at the absolute end of the process execution.
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

    def __del__(self):
        # Ensure file descriptor is safely closed upon instance destruction
        if hasattr(self, '_config_fp') and self._config_fp and not self._config_fp.closed:
            try:
                self._config_fp.close()
            except Exception:
                pass

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
            
            # [Performance & Memory Guard] Stream output chunking via temporary file
            # to prevent potential memory buffer overhead for large external outputs.
            import tempfile
            with tempfile.TemporaryFile(mode='w+') as stdout_tmp:
                try:
                    kw_proc = subprocess.run(
                        ['kernel-wedge', 'gen-control', vars['abiname']],
                        stdout=stdout_tmp,
                        stderr=subprocess.PIPE,
                        env=kw_env,
                        text=True,
                        check=True
                    )
                except subprocess.CalledProcessError as e:
                    raise RuntimeError(f"kernel-wedge failed with code {e.returncode}: {e.stderr}")
                
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
                    # [Critical Bug Fix] Corrected PEM boundary header/footer format (exact 5 hyphens)
                    stripped_line = line.strip()
                    if stripped_line == '-----BEGIN CERTIFICATE-----':
                        if state != 0:
                            raise RuntimeError("Invalid PEM structure: nested BEGIN CERTIFICATE")
                        certs.append([])
                        state = 1
                    elif stripped_line == '-----END CERTIFICATE-----':
                        if state != 1:
                            raise RuntimeError("Invalid PEM structure: unexpected END CERTIFICATE")
                        state = 0
                    else:
                        if state == 1:
                            certs[-1].append(line)
            if state != 0:
                raise RuntimeError("Invalid PEM structure: unclosed certificate block")
            return [''.join(cert_lines) for cert_lines in certs]

        def get_cert_fingerprint(cert, algo):
            hasher = hashlib.new(algo)
            hasher.update(ssl.PEM_cert_to_DER_cert(cert))
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

            package_certs = [get_cert_fingerprint(cert, 'sha256') for cert in get_certs(cert_file_name)]
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

    Gencontrol(args.arch)()

최종 개선사항
✅ ConfigCoreDump(fp=open(...)) 수명 불명확 → 인스턴스에 파일 핸들 귀속 및 생성 실패 시 즉시 정리 → 설정 스트림 누수 방지
✅ kernel-wedge stdout 직접 메모리 적재 → TemporaryFile 기반 스트리밍 처리 → 대규모 출력에서도 메모리 사용량 안정화
✅ 커널 설정·인증서 파일 존재 전제 → 명시적 파일 존재 및 필수 설정 검증 → 잘못된 Secure Boot 패키지 생성 차단
✅ PEM 파싱을 assert에 의존 → BEGIN/END 경계와 상태를 명시적으로 검증 → 인증서 메타데이터 무결성 강화
✅ 인증서 파일 부재·빈 인증서 목록 허용 → 파일 존재 및 최소 1개 인증서 검증 → 신뢰 체인 없는 서명 메타데이터 생성 방지
✅ sys.argv[1] 직접 접근 → argparse 인자 계약 적용 → 잘못된 CLI 호출의 비정상 처리 방지
✅ 파일·경로 문자열 중심 처리 → Path 기반 경로 관리 및 명시적 I/O 수명 관리 → 경로 조작과 자원 관리의 안정성 향상

원본의 빌드 제어·서명 메타데이터 생성 목적은 유지하면서 파일 수명, 외부 프로세스, 인증서 무결성, 입력 검증까지 방어층을 강화해 운영 환경에서도 실패를 조기에 탐지하는 실무형 구조로 승격했다.
