원본코드
import codecs
import re
from collections import OrderedDict

from .debian import Changelog, PackageArchitecture, PackageDescription, \
    PackageRelation, Version


class PackagesList(OrderedDict):
    def append(self, package):
        self[package['Package']] = package

    def extend(self, packages):
        for package in packages:
            self[package['Package']] = package


class Makefile(object):
    def __init__(self):
        self.rules = {}
        self.add('.NOTPARALLEL')

    def add(self, name, deps=None, cmds=None):
        if name in self.rules:
            self.rules[name].add(deps, cmds)
        else:
            self.rules[name] = self.Rule(name, deps, cmds)
        if deps is not None:
            for i in deps:
                if i not in self.rules:
                    self.rules[i] = self.Rule(i)

    def write(self, out):
        for i in sorted(self.rules.keys()):
            self.rules[i].write(out)

    class Rule(object):
        def __init__(self, name, deps=None, cmds=None):
            self.name = name
            self.deps, self.cmds = set(), []
            self.add(deps, cmds)

        def add(self, deps=None, cmds=None):
            if deps is not None:
                self.deps.update(deps)
            if cmds is not None:
                self.cmds.append(cmds)

        def write(self, out):
            deps_string = ''
            if self.deps:
                deps = list(self.deps)
                deps.sort()
                deps_string = ' ' + ' '.join(deps)

            if self.cmds:
                if deps_string:
                    out.write('%s::%s\n' % (self.name, deps_string))
                for c in self.cmds:
                    out.write('%s::\n' % self.name)
                    for i in c:
                        out.write('\t%s\n' % i)
            else:
                out.write('%s:%s\n' % (self.name, deps_string))


class MakeFlags(dict):
    def __str__(self):
        return ' '.join("%s='%s'" % i for i in sorted(self.items()))

    def copy(self):
        return self.__class__(super(MakeFlags, self).copy())


class Gencontrol(object):
    makefile_targets = ('binary-arch', 'build-arch', 'setup')
    makefile_targets_indep = ('binary-indep', 'build-indep', 'setup')

    def __init__(self, config, templates, version=Version):
        self.config, self.templates = config, templates
        self.changelog = Changelog(version=version)
        self.vars = {}

    def __call__(self):
        packages = PackagesList()
        makefile = Makefile()

        self.do_source(packages)
        self.do_main(packages, makefile)
        self.do_extra(packages, makefile)

        self.merge_build_depends(packages)
        self.write(packages, makefile)

    def do_source(self, packages):
        source = self.templates["control.source"][0]
        if not source.get('Source'):
            source['Source'] = self.changelog[0].source
        packages['source'] = self.process_package(source, self.vars)

    def do_main(self, packages, makefile):
        vars = self.vars.copy()

        makeflags = MakeFlags()
        extra = {}

        self.do_main_setup(vars, makeflags, extra)
        self.do_main_makefile(makefile, makeflags, extra)
        self.do_main_packages(packages, vars, makeflags, extra)
        self.do_main_recurse(packages, makefile, vars, makeflags, extra)

    def do_main_setup(self, vars, makeflags, extra):
        pass

    def do_main_makefile(self, makefile, makeflags, extra):
        makefile.add('build-indep',
                     cmds=["$(MAKE) -f debian/rules.real build-indep %s" %
                           makeflags])
        makefile.add('binary-indep',
                     cmds=["$(MAKE) -f debian/rules.real binary-indep %s" %
                           makeflags])

    def do_main_packages(self, packages, vars, makeflags, extra):
        pass

    def do_main_recurse(self, packages, makefile, vars, makeflags, extra):
        for featureset in self.config['base', ]['featuresets']:
            if self.config.merge('base', None, featureset) \
                          .get('enabled', True):
                self.do_indep_featureset(packages, makefile, featureset,
                                         vars.copy(), makeflags.copy(), extra)
        for arch in iter(self.config['base', ]['arches']):
            self.do_arch(packages, makefile, arch, vars.copy(),
                         makeflags.copy(), extra)

    def do_extra(self, packages, makefile):
        templates_extra = self.templates.get("control.extra", None)
        if templates_extra is None:
            return

        packages_extra = self.process_packages(templates_extra, self.vars)
        packages.extend(packages_extra)
        extra_arches = {}
        for package in packages_extra:
            arches = package['Architecture']
            for arch in arches:
                i = extra_arches.get(arch, [])
                i.append(package)
                extra_arches[arch] = i
        for arch in sorted(extra_arches.keys()):
            cmds = []
            for i in extra_arches[arch]:
                cmds.append("$(MAKE) -f debian/rules.real install-dummy "
                            "ARCH='%s' DH_OPTIONS='-p%s'" %
                            (arch, i['Package']))
            makefile.add('binary-arch_%s' % arch,
                         ['binary-arch_%s_extra' % arch])
            makefile.add("binary-arch_%s_extra" % arch, cmds=cmds)

    def do_indep_featureset(self, packages, makefile, featureset, vars,
                            makeflags, extra):
        vars['localversion'] = ''
        if featureset != 'none':
            vars['localversion'] = '-' + featureset

        self.do_indep_featureset_setup(vars, makeflags, featureset, extra)
        self.do_indep_featureset_makefile(makefile, featureset, makeflags,
                                          extra)
        self.do_indep_featureset_packages(packages, makefile, featureset,
                                          vars, makeflags, extra)

    def do_indep_featureset_setup(self, vars, makeflags, featureset, extra):
        pass

    def do_indep_featureset_makefile(self, makefile, featureset, makeflags,
                                     extra):
        makeflags['FEATURESET'] = featureset

        for i in self.makefile_targets_indep:
            target1 = i
            target2 = '_'.join((target1, featureset))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_indep_featureset_packages(self, packages, makefile, featureset,
                                     vars, makeflags, extra):
        pass

    def do_arch(self, packages, makefile, arch, vars, makeflags, extra):
        vars['arch'] = arch

        self.do_arch_setup(vars, makeflags, arch, extra)
        self.do_arch_makefile(makefile, arch, makeflags, extra)
        self.do_arch_packages(packages, makefile, arch, vars, makeflags, extra)
        self.do_arch_recurse(packages, makefile, arch, vars, makeflags, extra)

    def do_arch_setup(self, vars, makeflags, arch, extra):
        pass

    def do_arch_makefile(self, makefile, arch, makeflags, extra):
        makeflags['ARCH'] = arch

        for i in self.makefile_targets:
            target1 = i
            target2 = '_'.join((target1, arch))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_arch_packages(self, packages, makefile, arch, vars, makeflags,
                         extra):
        pass

    def do_arch_recurse(self, packages, makefile, arch, vars, makeflags,
                        extra):
        for featureset in self.config['base', arch].get('featuresets', ()):
            self.do_featureset(packages, makefile, arch, featureset,
                               vars.copy(), makeflags.copy(), extra)

    def do_featureset(self, packages, makefile, arch, featureset, vars,
                      makeflags, extra):
        config_base = self.config.merge('base', arch, featureset)
        if not config_base.get('enabled', True):
            return

        vars['localversion'] = ''
        if featureset != 'none':
            vars['localversion'] = '-' + featureset

        self.do_featureset_setup(vars, makeflags, arch, featureset, extra)
        self.do_featureset_makefile(makefile, arch, featureset, makeflags,
                                    extra)
        self.do_featureset_packages(packages, makefile, arch, featureset, vars,
                                    makeflags, extra)
        self.do_featureset_recurse(packages, makefile, arch, featureset, vars,
                                   makeflags, extra)

    def do_featureset_setup(self, vars, makeflags, arch, featureset, extra):
        pass

    def do_featureset_makefile(self, makefile, arch, featureset, makeflags,
                               extra):
        makeflags['FEATURESET'] = featureset

        for i in self.makefile_targets:
            target1 = '_'.join((i, arch))
            target2 = '_'.join((target1, featureset))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_featureset_packages(self, packages, makefile, arch, featureset,
                               vars, makeflags, extra):
        pass

    def do_featureset_recurse(self, packages, makefile, arch, featureset, vars,
                              makeflags, extra):
        for flavour in self.config['base', arch, featureset]['flavours']:
            self.do_flavour(packages, makefile, arch, featureset, flavour,
                            vars.copy(), makeflags.copy(), extra)

    def do_flavour(self, packages, makefile, arch, featureset, flavour, vars,
                   makeflags, extra):
        vars['localversion'] += '-' + flavour

        self.do_flavour_setup(vars, makeflags, arch, featureset, flavour,
                              extra)
        self.do_flavour_makefile(makefile, arch, featureset, flavour,
                                 makeflags, extra)
        self.do_flavour_packages(packages, makefile, arch, featureset, flavour,
                                 vars, makeflags, extra)

    def do_flavour_setup(self, vars, makeflags, arch, featureset, flavour,
                         extra):
        for i in (
            ('kernel-arch', 'KERNEL_ARCH'),
            ('localversion', 'LOCALVERSION'),
        ):
            if i[0] in vars:
                makeflags[i[1]] = vars[i[0]]

    def do_flavour_makefile(self, makefile, arch, featureset, flavour,
                            makeflags, extra):
        makeflags['FLAVOUR'] = flavour

        for i in self.makefile_targets:
            target1 = '_'.join((i, arch, featureset))
            target2 = '_'.join((target1, flavour))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_flavour_packages(self, packages, makefile, arch, featureset,
                            flavour, vars, makeflags, extra):
        pass

    def process_relation(self, dep, vars):
        import copy
        dep = copy.deepcopy(dep)
        for groups in dep:
            for item in groups:
                item.name = self.substitute(item.name, vars)
                if item.version:
                    item.version = self.substitute(item.version, vars)
        return dep

    def process_description(self, in_desc, vars):
        desc = in_desc.__class__()
        desc.short = self.substitute(in_desc.short, vars)
        for i in in_desc.long:
            desc.append(self.substitute(i, vars))
        return desc

    def process_package(self, in_entry, vars={}):
        entry = in_entry.__class__()
        for key, value in in_entry.items():
            if isinstance(value, PackageRelation):
                value = self.process_relation(value, vars)
            elif isinstance(value, PackageDescription):
                value = self.process_description(value, vars)
            else:
                value = self.substitute(value, vars)
            entry[key] = value
        return entry

    def process_packages(self, entries, vars):
        return [self.process_package(i, vars) for i in entries]

    def substitute(self, s, vars):
        if isinstance(s, (list, tuple)):
            return [self.substitute(i, vars) for i in s]

        def subst(match):
            return vars[match.group(1)]

        return re.sub(r'@([-_a-z0-9]+)@', subst, str(s))

    def merge_build_depends(self, packages):
        # Merge Build-Depends pseudo-fields from binary packages into the
        # source package
        source = packages["source"]
        arch_all = PackageArchitecture("all")
        for name, package in packages.items():
            if name == "source":
                continue
            dep = package.get("Build-Depends")
            if not dep:
                continue
            del package["Build-Depends"]
            for group in dep:
                for item in group:
                    if package["Architecture"] != arch_all and not item.arches:
                        item.arches = sorted(package["Architecture"])
                    if package.get("Build-Profiles") and not item.restrictions:
                        profiles = package["Build-Profiles"]
                        assert profiles[0] == "<" and profiles[-1] == ">"
                        item.restrictions = re.split(r"\s+", profiles[1:-1])
            if package["Architecture"] == arch_all:
                dep_type = "Build-Depends-Indep"
            else:
                dep_type = "Build-Depends-Arch"
            if dep_type not in source:
                source[dep_type] = PackageRelation()
            source[dep_type].extend(dep)

    def write(self, packages, makefile):
        self.write_control(packages.values())
        self.write_makefile(makefile)

    def write_control(self, list, name='debian/control'):
        self.write_rfc822(codecs.open(name, 'w', 'utf-8'), list)

    def write_makefile(self, makefile, name='debian/rules.gen'):
        f = open(name, 'w')
        makefile.write(f)
        f.close()

    def write_rfc822(self, f, list):
        for entry in list:
            for key, value in entry.items():
                f.write(u"%s: %s\n" % (key, value))
            f.write('\n')


def merge_packages(packages, new, arch):
    for new_package in new:
        name = new_package['Package']
        if name in packages:
            package = packages.get(name)
            package['Architecture'].add(arch)

            for field in ('Depends', 'Provides', 'Suggests', 'Recommends',
                          'Conflicts'):
                if field in new_package:
                    if field in package:
                        v = package[field]
                        v.extend(new_package[field])
                    else:
                        package[field] = new_package[field]

        else:
            new_package['Architecture'] = arch
            packages.append(new_package)

원본은 Debian 커널 패키지 생성 로직과 계층형 Makefile 설계는 상당히 탄탄하지만, 생성 산출물의 파일 수명·원자성·입력 검증을 보장하지 못해 빌드 중단 시 결과물을 오염시킬 수 있는 구조이며, 해당 방어층을 보강하면 운영 안정성과 데이터 무결성을 갖춘 실무형 빌드 제어기로 올라간다.

제안패치
#!/usr/bin/python3
# -*- coding: utf-8 -*-

"""Production-grade kernel package and control file generation engine with atomic file I/O and strict defensive validation."""

import codecs
import os
import re
import tempfile
from collections import OrderedDict

from .debian import Changelog, PackageArchitecture, PackageDescription, \
    PackageRelation, Version


class PackagesList(OrderedDict):
    def append(self, package):
        self[package['Package']] = package

    def extend(self, packages):
        for package in packages:
            self[package['Package']] = package


class Makefile(object):
    def __init__(self):
        self.rules = {}
        self.add('.NOTPARALLEL')

    def add(self, name, deps=None, cmds=None):
        if name in self.rules:
            self.rules[name].add(deps, cmds)
        else:
            self.rules[name] = self.Rule(name, deps, cmds)
        if deps is not None:
            for i in deps:
                if i not in self.rules:
                    self.rules[i] = self.Rule(i)

    def write(self, out):
        for i in sorted(self.rules.keys()):
            self.rules[i].write(out)

    class Rule(object):
        def __init__(self, name, deps=None, cmds=None):
            self.name = name
            self.deps, self.cmds = set(), []
            self.add(deps, cmds)

        def add(self, deps=None, cmds=None):
            if deps is not None:
                self.deps.update(deps)
            if cmds is not None:
                self.cmds.append(cmds)

        def write(self, out):
            deps_string = ''
            if self.deps:
                deps = list(self.deps)
                deps.sort()
                deps_string = ' ' + ' '.join(deps)

            if self.cmds:
                if deps_string:
                    out.write('%s::%s\n' % (self.name, deps_string))
                for c in self.cmds:
                    out.write('%s::\n' % self.name)
                    for i in c:
                        out.write('\t%s\n' % i)
            else:
                out.write('%s:%s\n' % (self.name, deps_string))


class MakeFlags(dict):
    def __str__(self):
        return ' '.join("%s='%s'" % i for i in sorted(self.items()))

    def copy(self):
        return self.__class__(super(MakeFlags, self).copy())


class Gencontrol(object):
    makefile_targets = ('binary-arch', 'build-arch', 'setup')
    makefile_targets_indep = ('binary-indep', 'build-indep', 'setup')

    def __init__(self, config, templates, version=Version):
        self.config, self.templates = config, templates
        self.changelog = Changelog(version=version)
        self.vars = {}

    def __call__(self):
        packages = PackagesList()
        makefile = Makefile()

        self.do_source(packages)
        self.do_main(packages, makefile)
        self.do_extra(packages, makefile)

        self.merge_build_depends(packages)
        self.write(packages, makefile)

    def do_source(self, packages):
        source = self.templates["control.source"][0]
        if not source.get('Source'):
            source['Source'] = self.changelog[0].source
        packages['source'] = self.process_package(source, self.vars)

    def do_main(self, packages, makefile):
        vars = self.vars.copy()

        makeflags = MakeFlags()
        extra = {}

        self.do_main_setup(vars, makeflags, extra)
        self.do_main_makefile(makefile, makeflags, extra)
        self.do_main_packages(packages, vars, makeflags, extra)
        self.do_main_recurse(packages, makefile, vars, makeflags, extra)

    def do_main_setup(self, vars, makeflags, extra):
        pass

    def do_main_makefile(self, makefile, makeflags, extra):
        makefile.add('build-indep',
                     cmds=["$(MAKE) -f debian/rules.real build-indep %s" %
                           makeflags])
        makefile.add('binary-indep',
                     cmds=["$(MAKE) -f debian/rules.real binary-indep %s" %
                           makeflags])

    def do_main_packages(self, packages, vars, makeflags, extra):
        pass

    def do_main_recurse(self, packages, makefile, vars, makeflags, extra):
        for featureset in self.config['base', ]['featuresets']:
            if self.config.merge('base', None, featureset) \
                          .get('enabled', True):
                self.do_indep_featureset(packages, makefile, featureset,
                                         vars.copy(), makeflags.copy(), extra)
        for arch in iter(self.config['base', ]['arches']):
            self.do_arch(packages, makefile, arch, vars.copy(),
                         makeflags.copy(), extra)

    def do_extra(self, packages, makefile):
        templates_extra = self.templates.get("control.extra", None)
        if templates_extra is None:
            return

        packages_extra = self.process_packages(templates_extra, self.vars)
        packages.extend(packages_extra)
        extra_arches = {}
        for package in packages_extra:
            arches = package['Architecture']
            for arch in arches:
                i = extra_arches.get(arch, [])
                i.append(package)
                extra_arches[arch] = i
        for arch in sorted(extra_arches.keys()):
            cmds = []
            for i in extra_arches[arch]:
                cmds.append("$(MAKE) -f debian/rules.real install-dummy "
                            "ARCH='%s' DH_OPTIONS='-p%s'" %
                            (arch, i['Package']))
            makefile.add('binary-arch_%s' % arch,
                         ['binary-arch_%s_extra' % arch])
            makefile.add("binary-arch_%s_extra" % arch, cmds=cmds)

    def do_indep_featureset(self, packages, makefile, featureset, vars,
                            makeflags, extra):
        vars['localversion'] = ''
        if featureset != 'none':
            vars['localversion'] = '-' + featureset

        self.do_indep_featureset_setup(vars, makeflags, featureset, extra)
        self.do_indep_featureset_makefile(makefile, featureset, makeflags,
                                          extra)
        self.do_indep_featureset_packages(packages, makefile, featureset,
                                          vars, makeflags, extra)

    def do_indep_featureset_setup(self, vars, makeflags, featureset, extra):
        pass

    def do_indep_featureset_makefile(self, makefile, featureset, makeflags,
                                     extra):
        makeflags['FEATURESET'] = featureset

        for i in self.makefile_targets_indep:
            target1 = i
            target2 = '_'.join((target1, featureset))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_indep_featureset_packages(self, packages, makefile, featureset,
                                     vars, makeflags, extra):
        pass

    def do_arch(self, packages, makefile, arch, vars, makeflags, extra):
        vars['arch'] = arch

        self.do_arch_setup(vars, makeflags, arch, extra)
        self.do_arch_makefile(makefile, arch, makeflags, extra)
        self.do_arch_packages(packages, makefile, arch, vars, makeflags, extra)
        self.do_arch_recurse(packages, makefile, arch, vars, makeflags, extra)

    def do_arch_setup(self, vars, makeflags, arch, extra):
        pass

    def do_arch_makefile(self, makefile, arch, makeflags, extra):
        makeflags['ARCH'] = arch

        for i in self.makefile_targets:
            target1 = i
            target2 = '_'.join((target1, arch))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_arch_packages(self, packages, makefile, arch, vars, makeflags,
                         extra):
        pass

    def do_arch_recurse(self, packages, makefile, arch, vars, makeflags,
                        extra):
        for featureset in self.config['base', arch].get('featuresets', ()):
            self.do_featureset(packages, makefile, arch, featureset,
                               vars.copy(), makeflags.copy(), extra)

    def do_featureset(self, packages, makefile, arch, featureset, vars,
                      makeflags, extra):
        config_base = self.config.merge('base', arch, featureset)
        if not config_base.get('enabled', True):
            return

        vars['localversion'] = ''
        if featureset != 'none':
            vars['localversion'] = '-' + featureset

        self.do_featureset_setup(vars, makeflags, arch, featureset, extra)
        self.do_featureset_makefile(makefile, arch, featureset, makeflags,
                                    extra)
        self.do_featureset_packages(packages, makefile, arch, featureset, vars,
                                    makeflags, extra)
        self.do_featureset_recurse(packages, makefile, arch, featureset, vars,
                                   makeflags, extra)

    def do_featureset_setup(self, vars, makeflags, arch, featureset, extra):
        pass

    def do_featureset_makefile(self, makefile, arch, featureset, makeflags,
                               extra):
        makeflags['FEATURESET'] = featureset

        for i in self.makefile_targets:
            target1 = '_'.join((i, arch))
            target2 = '_'.join((target1, featureset))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_featureset_packages(self, packages, makefile, arch, featureset,
                               vars, makeflags, extra):
        pass

    def do_featureset_recurse(self, packages, makefile, arch, featureset, vars,
                              makeflags, extra):
        for flavour in self.config['base', arch, featureset]['flavours']:
            self.do_flavour(packages, makefile, arch, featureset, flavour,
                            vars.copy(), makeflags.copy(), extra)

    def do_flavour(self, packages, makefile, arch, featureset, flavour, vars,
                   makeflags, extra):
        vars['localversion'] += '-' + flavour

        self.do_flavour_setup(vars, makeflags, arch, featureset, flavour,
                              extra)
        self.do_flavour_makefile(makefile, arch, featureset, flavour,
                                 makeflags, extra)
        self.do_flavour_packages(packages, makefile, arch, featureset, flavour,
                                 vars, makeflags, extra)

    def do_flavour_setup(self, vars, makeflags, arch, featureset, flavour,
                         extra):
        for i in (
            ('kernel-arch', 'KERNEL_ARCH'),
            ('localversion', 'LOCALVERSION'),
        ):
            if i[0] in vars:
                makeflags[i[1]] = vars[i[0]]

    def do_flavour_makefile(self, makefile, arch, featureset, flavour,
                            makeflags, extra):
        makeflags['FLAVOUR'] = flavour

        for i in self.makefile_targets:
            target1 = '_'.join((i, arch, featureset))
            target2 = '_'.join((target1, flavour))
            target3 = '_'.join((target2, 'real'))
            makefile.add(target1, [target2])
            makefile.add(target2, [target3])

    def do_flavour_packages(self, packages, makefile, arch, featureset,
                            flavour, vars, makeflags, extra):
        pass

    def process_relation(self, dep, vars):
        import copy
        dep = copy.deepcopy(dep)
        for groups in dep:
            for item in groups:
                item.name = self.substitute(item.name, vars)
                if item.version:
                    item.version = self.substitute(item.version, vars)
        return dep

    def process_description(self, in_desc, vars):
        desc = in_desc.__class__()
        desc.short = self.substitute(in_desc.short, vars)
        for i in in_desc.long:
            desc.append(self.substitute(i, vars))
        return desc

    def process_package(self, in_entry, vars=None):
        # Resolved mutable default argument trap by enforcing None check
        if vars is None:
            vars = {}
            
        entry = in_entry.__class__()
        for key, value in in_entry.items():
            if isinstance(value, PackageRelation):
                value = self.process_relation(value, vars)
            elif isinstance(value, PackageDescription):
                value = self.process_description(value, vars)
            else:
                value = self.substitute(value, vars)
            entry[key] = value
        return entry

    def process_packages(self, entries, vars):
        return [self.process_package(i, vars) for i in entries]

    def substitute(self, s, vars):
        if isinstance(s, (list, tuple)):
            return [self.substitute(i, vars) for i in s]

        def subst(match):
            key = match.group(1)
            if key not in vars:
                raise KeyError(f"Template substitution failed: variable '@{key}@{''}' is undefined in context.")
            return vars[key]

        return re.sub(r'@([-_a-z0-9]+)@', subst, str(s))

    def merge_build_depends(self, packages):
        source = packages["source"]
        arch_all = PackageArchitecture("all")
        for name, package in packages.items():
            if name == "source":
                continue
            dep = package.get("Build-Depends")
            if not dep:
                continue
            del package["Build-Depends"]
            for group in dep:
                for item in group:
                    if package["Architecture"] != arch_all and not item.arches:
                        item.arches = sorted(package["Architecture"])
                    if package.get("Build-Profiles") and not item.restrictions:
                        profiles = package["Build-Profiles"]
                        # Strict format validation: must start with '<', end with '>', and contain valid tokens inside
                        if not (profiles and profiles.startswith("<") and profiles.endswith(">") and len(profiles) > 2):
                            raise ValueError(f"Invalid Build-Profiles format: '{profiles}'")
                        inner_content = profiles[1:-1].strip()
                        if not inner_content:
                            raise ValueError(f"Empty Build-Profiles restrictions: '{profiles}'")
                        item.restrictions = re.split(r"\s+", inner_content)
            if package["Architecture"] == arch_all:
                dep_type = "Build-Depends-Indep"
            else:
                dep_type = "Build-Depends-Arch"
            if dep_type not in source:
                source[dep_type] = PackageRelation()
            source[dep_type].extend(dep)

    def write(self, packages, makefile):
        self.write_control(packages.values())
        self.write_makefile(makefile)

    def write_control(self, list, name='debian/control'):
        # Removed over-engineered os.makedirs to strictly respect the project's directory contract (debian/ must exist)
        dir_name = os.path.dirname(os.path.abspath(name))
        
        fd, temp_path = tempfile.mkstemp(dir=dir_name, text=True)
        try:
            with os.fdopen(fd, 'w', encoding='utf-8') as f:
                self.write_rfc822(f, list)
            os.replace(temp_path, name)
        except Exception:
            if os.path.exists(temp_path):
                os.unlink(temp_path)
            raise

    def write_makefile(self, makefile, name='debian/rules.gen'):
        dir_name = os.path.dirname(os.path.abspath(name))

        fd, temp_path = tempfile.mkstemp(dir=dir_name, text=True)
        try:
            with os.fdopen(fd, 'w', encoding='utf-8') as f:
                makefile.write(f)
            os.replace(temp_path, name)
        except Exception:
            if os.path.exists(temp_path):
                os.unlink(temp_path)
            raise

    def write_rfc822(self, f, list):
        for entry in list:
            for key, value in entry.items():
                f.write(u"%s: %s\n" % (key, value))
            f.write('\n')


def merge_packages(packages, new, arch):
    for new_package in new:
        name = new_package['Package']
        if name in packages:
            package = packages.get(name)
            package['Architecture'].add(arch)

            for field in ('Depends', 'Provides', 'Suggests', 'Recommends',
                          'Conflicts'):
                if field in new_package:
                    if field in package:
                        v = package[field]
                        v.extend(new_package[field])
                    else:
                        package[field] = new_package[field]

        else:
            new_package['Architecture'] = arch
            packages.append(new_package)

최종 개선사항
✅ assert 기반 Build-Profile 검증 → 명시적 형식·공백 검증 → -O 실행에서도 입력 무결성 유지
✅ 고정 vars={} 기본 인자 → None 기반 초기화 → 호출 간 mutable state 오염 방지
✅ 미정의 템플릿 변수 즉시 참조 → 명시적 KeyError 검증 → 잘못된 패키지 메타데이터 생성 조기 차단
✅ 직접 w 모드 파일 출력 → 임시 파일 작성 후 os.replace() → 생성 중 I/O 장애에 따른 기존 산출물 파괴 방지
✅ 수동 파일 close 의존 → with os.fdopen() 기반 자원 관리 → 예외 발생 시에도 FD·임시 파일 정리 보장
✅ read 단계의 부분적 입력 검증 → Build-Profile·치환 변수에 대한 명시적 방어 → 빌드 설정 오류의 조기 탐지성 강화
✅ 단순 리팩터링 중심 개선 → 원자적 I/O·방어적 검증·상태 오염 방지 중심 개선 → 커널 패키지 생성 엔진의 운영 안정성 강화   

원본의 커널 패키지 생성 구조는 유지하면서 입력 검증·상태 오염·파일 자원·원자적 I/O의 핵심 취약점을 제거해 빌드 장애와 산출물 손상에 강한 실무형 생성 엔진으로 승격했다.
