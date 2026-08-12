원본코드
#!/usr/bin/python3

import sys
import glob
import os
import re

from debian_linux.abi import Symbols
from debian_linux.config import ConfigCoreDump
from debian_linux.debian import Changelog, VersionLinux


class CheckAbi(object):
    class SymbolInfo(object):
        def __init__(self, symbol, symbol_ref=None):
            self.symbol = symbol
            self.symbol_ref = symbol_ref or symbol

        @property
        def module(self):
            return self.symbol.module

        @property
        def name(self):
            return self.symbol.name

        def write(self, out, ignored):
            info = []
            if ignored:
                info.append("ignored")
            for name in ('module', 'version', 'export'):
                data = getattr(self.symbol, name)
                data_ref = getattr(self.symbol_ref, name)
                if data != data_ref:
                    info.append("%s: %s -> %s" % (name, data_ref, data))
                else:
                    info.append("%s: %s" % (name, data))
            out.write("%-48s %s\n" % (self.symbol.name, ", ".join(info)))

    def __init__(self, config, dir, arch, featureset, flavour):
        self.config = config
        self.arch, self.featureset, self.flavour = arch, featureset, flavour

        self.filename_new = "%s/Module.symvers" % dir

        try:
            version_abi = (self.config[('version',)]['abiname_base'] + '-'
                           + self.config['abi', arch]['abiname'])
        except KeyError:
            version_abi = self.config[('version',)]['abiname']
        self.filename_ref = ("debian/abi/%s/%s_%s_%s" %
                             (version_abi, arch, featureset, flavour))

    def __call__(self, out):
        ret = 0

        new = Symbols(open(self.filename_new))
        unversioned = [name for name in new
                       if new[name].version == '0x00000000']
        if unversioned:
            out.write("ABI is not completely versioned!  "
                      "Refusing to continue.\n")
            out.write("\nUnversioned symbols:\n")
            for name in sorted(unversioned):
                self.SymbolInfo(new[name]).write(out, False)
            ret = 1

        try:
            ref = Symbols(open(self.filename_ref))
        except IOError:
            out.write("Can't read ABI reference.  ABI not checked!\n")
            return ret

        symbols, add, change, remove = self._cmp(ref, new)

        ignore = self._ignore(symbols)

        add_effective = add - ignore
        change_effective = change - ignore
        remove_effective = remove - ignore

        if change_effective or remove_effective:
            out.write("ABI has changed!  Refusing to continue.\n")
            ret = 1
        elif change or remove:
            out.write("ABI has changed but all changes have been ignored.  "
                      "Continuing.\n")
        elif add_effective:
            out.write("New symbols have been added.  Continuing.\n")
        elif add:
            out.write("New symbols have been added but have been ignored.  "
                      "Continuing.\n")
        else:
            out.write("No ABI changes.\n")

        if add:
            out.write("\nAdded symbols:\n")
            for name in sorted(add):
                symbols[name].write(out, name in ignore)

        if change:
            out.write("\nChanged symbols:\n")
            for name in sorted(change):
                symbols[name].write(out, name in ignore)

        if remove:
            out.write("\nRemoved symbols:\n")
            for name in sorted(remove):
                symbols[name].write(out, name in ignore)

        return ret

    def _cmp(self, ref, new):
        ref_names = set(ref.keys())
        new_names = set(new.keys())

        add = set()
        change = set()
        remove = set()

        symbols = {}

        for name in new_names - ref_names:
            add.add(name)
            symbols[name] = self.SymbolInfo(new[name])

        for name in ref_names.intersection(new_names):
            s_ref = ref[name]
            s_new = new[name]

            if s_ref != s_new:
                change.add(name)
                symbols[name] = self.SymbolInfo(s_new, s_ref)

        for name in ref_names - new_names:
            remove.add(name)
            symbols[name] = self.SymbolInfo(ref[name])

        return symbols, add, change, remove

    def _ignore_pattern(self, pattern):
        ret = []
        for i in re.split(r'(\*\*?)', pattern):
            if i == '*':
                ret.append(r'[^/]*')
            elif i == '**':
                ret.append(r'.*')
            elif i:
                ret.append(re.escape(i))
        return re.compile('^' + ''.join(ret) + '$')

    def _ignore(self, symbols):
        # TODO: let config merge this lists
        configs = []
        configs.append(self.config.get(('abi', self.arch, self.featureset,
                                        self.flavour), {}))
        configs.append(self.config.get(('abi', self.arch, None, self.flavour),
                                       {}))
        configs.append(self.config.get(('abi', self.arch, self.featureset),
                                       {}))
        configs.append(self.config.get(('abi', self.arch), {}))
        configs.append(self.config.get(('abi', None, self.featureset), {}))
        configs.append(self.config.get(('abi',), {}))

        ignores = set()
        for config in configs:
            ignores.update(config.get('ignore-changes', []))

        filtered = set()
        for ignore in ignores:
            type = 'name'
            if ':' in ignore:
                type, ignore = ignore.split(':')
            if type in ('name', 'module'):
                p = self._ignore_pattern(ignore)
                for symbol in symbols.values():
                    if p.match(getattr(symbol, type)):
                        filtered.add(symbol.name)
            else:
                raise NotImplementedError

        return filtered


class CheckImage(object):
    def __init__(self, config, dir, arch, featureset, flavour):
        self.dir = dir
        self.arch, self.featureset, self.flavour = arch, featureset, flavour

        self.changelog = Changelog(version=VersionLinux)[0]

        self.config_entry_base = config.merge('base', arch, featureset,
                                              flavour)
        self.config_entry_build = config.merge('build', arch, featureset,
                                               flavour)
        self.config_entry_image = config.merge('image', arch, featureset,
                                               flavour)

    def __call__(self, out):
        image = self.config_entry_build.get('image-file')
        uncompressed_image = self.config_entry_build \
                                 .get('uncompressed-image-file')

        if not image:
            # TODO: Bail out
            return 0

        image = os.path.join(self.dir, image)
        if uncompressed_image:
            uncompressed_image = os.path.join(self.dir, uncompressed_image)

        fail = 0

        fail |= self.check_size(out, image, uncompressed_image)

        return fail

    def check_size(self, out, image, uncompressed_image):
        value = self.config_entry_image.get('check-size')

        if not value:
            return 0

        dtb_size = 0
        if self.config_entry_image.get('check-size-with-dtb'):
            for dtb in glob.glob(
                    os.path.join(self.dir, 'arch',
                                 self.config_entry_base['kernel-arch'],
                                 'boot/dts/*.dtb')):
                dtb_size = max(dtb_size, os.stat(dtb).st_size)

        size = os.stat(image).st_size + dtb_size

        # 1% overhead is desirable in order to cope with growth
        # through the lifetime of a stable release. Warn if this is
        # not the case.
        usage = (float(size)/value) * 100.0
        out.write('Image size %d/%d, using %.2f%%.  ' % (size, value, usage))
        if size > value:
            out.write('Too large.  Refusing to continue.\n')
            return 1
        elif usage >= 99.0:
            out.write('Under 1%% space in %s.  ' % self.changelog.distribution)
        else:
            out.write('Image fits.  ')
        out.write('Continuing.\n')

        # Also check the uncompressed image
        if uncompressed_image and \
           self.config_entry_image.get('check-uncompressed-size'):
            value = self.config_entry_image.get('check-uncompressed-size')
            size = os.stat(uncompressed_image).st_size
            usage = (float(size)/value) * 100.0
            out.write('Uncompressed Image size %d/%d, using %.2f%%.  ' %
                      (size, value, usage))
            if size > value:
                out.write('Too large.  Refusing to continue.\n')
                return 1
            elif usage >= 99.0:
                out.write('Uncompressed Image Under 1%% space in %s.  ' %
                          self.changelog.distribution)
            else:
                out.write('Uncompressed Image fits.  ')
            out.write('Continuing.\n')

        return 0


class Main(object):
    def __init__(self, dir, arch, featureset, flavour):
        self.args = dir, arch, featureset, flavour

        self.config = ConfigCoreDump(open("debian/config.defines.dump", "rb"))

    def __call__(self):
        fail = 0

        for c in CheckAbi, CheckImage:
            fail |= c(self.config, *self.args)(sys.stdout)

        return fail


if __name__ == '__main__':
    sys.exit(Main(*sys.argv[1:])())

원본은 ABI 변경과 커널 이미지 크기를 검사하는 핵심 방어선 역할은 충실하지만, 파일 자원 관리·검증 실패의 성공 처리·입력 및 파일 상태 검증이 부족해 빌드 무결성을 오판할 여지가 있는 운영 취약형 검증 코드다.

제안패치
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""Kernel ABI and image-size build validation."""

import argparse
import glob
import os
import re
import sys
from pathlib import Path

from debian_linux.abi import Symbols
from debian_linux.config import ConfigCoreDump
from debian_linux.debian import Changelog, VersionLinux


class CheckAbi:
    class SymbolInfo:
        __slots__ = ("symbol", "symbol_ref")

        def __init__(self, symbol, symbol_ref=None):
            self.symbol = symbol
            self.symbol_ref = symbol_ref or symbol

        @property
        def module(self):
            return self.symbol.module

        @property
        def name(self):
            return self.symbol.name

        def write(self, out, ignored):
            info = ["ignored"] if ignored else []

            for name in ("module", "version", "export"):
                current = getattr(self.symbol, name)
                reference = getattr(self.symbol_ref, name)

                if current != reference:
                    info.append(f"{name}: {reference} -> {current}")
                else:
                    info.append(f"{name}: {current}")

            out.write(
                f"{self.symbol.name:<48} {', '.join(info)}\n"
            )

    def __init__(self, config, directory, arch, featureset, flavour):
        self.config = config
        self.arch = arch
        self.featureset = featureset
        self.flavour = flavour

        self.filename_new = Path(directory) / "Module.symvers"

        try:
            version_abi = (
                self.config[("version",)]["abiname_base"]
                + "-"
                + self.config["abi", arch]["abiname"]
            )
        except KeyError:
            version_abi = self.config[("version",)]["abiname"]

        self.filename_ref = (
            Path("debian/abi")
            / version_abi
            / f"{arch}_{featureset}_{flavour}"
        )

    def __call__(self, out):
        ret = 0

        if not self.filename_new.is_file():
            out.write(
                f"Can't read generated ABI file: {self.filename_new}\n"
            )
            return 1

        with self.filename_new.open("r") as stream:
            new = Symbols(stream)

        unversioned = [
            name
            for name in new
            if new[name].version == "0x00000000"
        ]

        if unversioned:
            out.write(
                "ABI is not completely versioned! "
                "Refusing to continue.\n"
            )
            out.write("\nUnversioned symbols:\n")

            for name in sorted(unversioned):
                self.SymbolInfo(new[name]).write(out, False)

            ret = 1

        if not self.filename_ref.is_file():
            out.write(
                f"Can't read ABI reference: {self.filename_ref}\n"
            )
            out.write("ABI cannot be checked. Refusing to continue.\n")
            return 1

        with self.filename_ref.open("r") as stream:
            ref = Symbols(stream)

        symbols, added, changed, removed = self._cmp(ref, new)
        ignored = self._ignore(symbols)

        effective_changed = changed - ignored
        effective_removed = removed - ignored

        if effective_changed or effective_removed:
            out.write(
                "ABI has changed! Refusing to continue.\n"
            )
            ret = 1
        elif changed or removed:
            out.write(
                "ABI has changed but all changes have been ignored. "
                "Continuing.\n"
            )
        elif added - ignored:
            out.write(
                "New symbols have been added. Continuing.\n"
            )
        elif added:
            out.write(
                "New symbols have been added but have been ignored. "
                "Continuing.\n"
            )
        else:
            out.write("No ABI changes.\n")

        for title, collection in (
            ("Added", added),
            ("Changed", changed),
            ("Removed", removed),
        ):
            if not collection:
                continue

            out.write(f"\n{title} symbols:\n")

            for name in sorted(collection):
                symbols[name].write(out, name in ignored)

        return ret

    def _cmp(self, ref, new):
        ref_names = set(ref)
        new_names = set(new)

        added = new_names - ref_names
        removed = ref_names - new_names
        changed = {
            name
            for name in ref_names & new_names
            if ref[name] != new[name]
        }

        symbols = {}

        for name in added:
            symbols[name] = self.SymbolInfo(new[name])

        for name in changed:
            symbols[name] = self.SymbolInfo(new[name], ref[name])

        for name in removed:
            symbols[name] = self.SymbolInfo(ref[name])

        return symbols, added, changed, removed

    def _ignore_pattern(self, pattern):
        parts = []

        for part in re.split(r"(\*\*?)", pattern):
            if part == "*":
                parts.append(r"[^/]*")
            elif part == "**":
                parts.append(r".*")
            elif part:
                parts.append(re.escape(part))

        return re.compile("^" + "".join(parts) + "$")

    def _ignore(self, symbols):
        configs = (
            self.config.get(
                ("abi", self.arch, self.featureset, self.flavour), {}
            ),
            self.config.get(
                ("abi", self.arch, None, self.flavour), {}
            ),
            self.config.get(
                ("abi", self.arch, self.featureset), {}
            ),
            self.config.get(("abi", self.arch), {}),
            self.config.get(("abi", None, self.featureset), {}),
            self.config.get(("abi",), {}),
        )

        patterns = set()

        for config in configs:
            patterns.update(config.get("ignore-changes", []))

        ignored = set()

        for expression in patterns:
            kind = "name"

            if ":" in expression:
                kind, expression = expression.split(":", 1)

            if kind not in ("name", "module"):
                raise ValueError(
                    f"Unsupported ABI ignore type: {kind}"
                )

            pattern = self._ignore_pattern(expression)

            for symbol in symbols.values():
                if pattern.match(getattr(symbol, kind)):
                    ignored.add(symbol.name)

        return ignored


class CheckImage:
    def __init__(self, config, directory, arch, featureset, flavour):
        self.dir = Path(directory)
        self.arch = arch
        self.featureset = featureset
        self.flavour = flavour

        self.changelog = Changelog(version=VersionLinux)[0]

        self.config_entry_base = config.merge(
            "base", arch, featureset, flavour
        )
        self.config_entry_build = config.merge(
            "build", arch, featureset, flavour
        )
        self.config_entry_image = config.merge(
            "image", arch, featureset, flavour
        )

    def __call__(self, out):
        image_name = self.config_entry_build.get("image-file")
        uncompressed_name = self.config_entry_build.get(
            "uncompressed-image-file"
        )

        if not image_name:
            return 0

        image = self.dir / image_name
        uncompressed = (
            self.dir / uncompressed_name
            if uncompressed_name
            else None
        )

        if not image.is_file():
            out.write(f"Kernel image not found: {image}\n")
            return 1

        if uncompressed is not None:
            if not uncompressed.is_file():
                out.write(
                    f"Uncompressed kernel image not found: "
                    f"{uncompressed}\n"
                )
                return 1

        return self.check_size(out, image, uncompressed)

    def check_size(self, out, image, uncompressed_image):
        limit = self.config_entry_image.get("check-size")

        if not limit:
            return 0

        dtb_size = 0

        if self.config_entry_image.get("check-size-with-dtb"):
            kernel_arch = self.config_entry_base.get("kernel-arch")

            if not kernel_arch:
                out.write(
                    "Kernel architecture is missing; "
                    "cannot validate DTB size.\n"
                )
                return 1

            pattern = (
                self.dir
                / "arch"
                / kernel_arch
                / "boot"
                / "dts"
                / "*.dtb"
            )

            dtbs = glob.glob(str(pattern))

            for dtb in dtbs:
                try:
                    dtb_size = max(
                        dtb_size,
                        Path(dtb).stat().st_size,
                    )
                except OSError as exc:
                    out.write(
                        f"Failed to stat DTB {dtb}: {exc}\n"
                    )
                    return 1

        try:
            size = image.stat().st_size + dtb_size
        except OSError as exc:
            out.write(f"Failed to stat kernel image: {exc}\n")
            return 1

        usage = size * 100.0 / limit

        out.write(
            f"Image size {size}/{limit}, using {usage:.2f}%.  "
        )

        if size > limit:
            out.write("Too large. Refusing to continue.\n")
            return 1

        if usage >= 99.0:
            out.write(
                f"Under 1%% space in {self.changelog.distribution}. "
            )
        else:
            out.write("Image fits. ")

        out.write("Continuing.\n")

        uncompressed_limit = self.config_entry_image.get(
            "check-uncompressed-size"
        )

        if uncompressed_image and uncompressed_limit:
            try:
                uncompressed_size = (
                    uncompressed_image.stat().st_size
                )
            except OSError as exc:
                out.write(
                    f"Failed to stat uncompressed image: {exc}\n"
                )
                return 1

            usage = (
                uncompressed_size * 100.0 / uncompressed_limit
            )

            out.write(
                f"Uncompressed Image size "
                f"{uncompressed_size}/{uncompressed_limit}, "
                f"using {usage:.2f}%.  "
            )

            if uncompressed_size > uncompressed_limit:
                out.write(
                    "Too large. Refusing to continue.\n"
                )
                return 1

            if usage >= 99.0:
                out.write(
                    f"Uncompressed Image Under 1%% space in "
                    f"{self.changelog.distribution}. "
                )
            else:
                out.write(
                    "Uncompressed Image fits. "
                )

            out.write("Continuing.\n")

        return 0


class Main:
    def __init__(self, directory, arch, featureset, flavour):
        self.args = (directory, arch, featureset, flavour)

        dump_path = Path("debian/config.defines.dump")

        if not dump_path.is_file():
            raise RuntimeError(
                f"Configuration dump not found: {dump_path}"
            )

        # ConfigCoreDump가 내부적으로 fp를 보유할 가능성이 있으므로
        # 객체 생존 기간 동안 스트림도 유지한다.
        self.config_file = dump_path.open("rb")
        try:
            self.config = ConfigCoreDump(fp=self.config_file)
        except Exception:
            self.config_file.close()
            raise

    def __call__(self):
        try:
            fail = 0

            for checker in (CheckAbi, CheckImage):
                result = checker(
                    self.config,
                    *self.args,
                )(sys.stdout)

                fail |= result

            return fail
        finally:
            self.config_file.close()


def main():
    parser = argparse.ArgumentParser(
        description="Check kernel ABI and image size compliance."
    )

    parser.add_argument("dir", help="Build directory")
    parser.add_argument("arch", help="Target architecture")
    parser.add_argument("featureset", help="Kernel featureset")
    parser.add_argument("flavour", help="Kernel flavour")

    args = parser.parse_args()

    return Main(
        args.dir,
        args.arch,
        args.featureset,
        args.flavour,
    )()


if __name__ == "__main__":
    sys.exit(main())

최종 개선사항
✅ 파일 수명 단절 가능성 → ConfigCoreDump와 스트림 lifetime 동기화 → 런타임 설정 조회 안정성 확보
✅ ABI 참조 누락을 성공으로 처리 → 검증 불능을 명시적 실패로 전환 → False Success 차단
✅ 이미지·압축 이미지 존재성 미검증 → 사전 파일 상태 검증 → 빌드 검증 신뢰성 강화
✅ DTB stat() 실패 무시 → 오류 즉시 실패 처리 → 실제 이미지 크기 누락에 따른 오판 방지
✅ 파일 핸들 직접 생성·방치 → 명시적 lifecycle 관리 → 파일 디스크립터 누수 방지
✅ 구형 CLI 검증 및 무방비 인자 처리 → argparse 기반 명시적 계약 → 잘못된 빌드 인자 조기 차단
✅ ABI 비교 로직의 반복 분기 → 집합 연산과 명시적 결과 분류 → 비교 로직의 정확성과 유지보수성 향상

원본의 단순 빌드 체크 수준에서 검증 불능과 데이터 누락을 성공으로 오판하지 않는 방어형 검증기로 승격했으며, 핵심은 기능 추가가 아니라 ABI 무결성과 빌드 판정의 신뢰성을 끝까지 보존하는 구조다.    
