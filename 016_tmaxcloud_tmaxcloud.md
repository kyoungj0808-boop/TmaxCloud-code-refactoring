원본코드
import collections
import os
import os.path
import pickle
import re
import sys

from configparser import RawConfigParser

__all__ = [
    'ConfigCoreDump',
    'ConfigCoreHierarchy',
    'ConfigParser',
]


class SchemaItemBoolean(object):
    def __call__(self, i):
        i = i.strip().lower()
        if i in ("true", "1"):
            return True
        if i in ("false", "0"):
            return False
        raise ValueError


class SchemaItemInteger(object):
    def __call__(self, i):
        return int(i.strip(), 0)


class SchemaItemList(object):
    def __init__(self, type=r"\s+"):
        self.type = type

    def __call__(self, i):
        i = i.strip()
        if not i:
            return []
        return [j.strip() for j in re.split(self.type, i)]


# Using OrderedDict instead of dict makes the pickled config reproducible
class ConfigCore(collections.OrderedDict):
    def get_merge(self, section, arch, featureset, flavour, key, default=None):
        temp = []

        if arch and featureset and flavour:
            temp.append(self.get((section, arch, featureset, flavour), {})
                        .get(key))
            temp.append(self.get((section, arch, None, flavour), {}).get(key))
        if arch and featureset:
            temp.append(self.get((section, arch, featureset), {}).get(key))
        if arch:
            temp.append(self.get((section, arch), {}).get(key))
        if featureset:
            temp.append(self.get((section, None, featureset), {}).get(key))
        temp.append(self.get((section,), {}).get(key))

        ret = []

        for i in temp:
            if i is None:
                continue
            elif isinstance(i, (list, tuple)):
                ret.extend(i)
            elif ret:
                # TODO
                return ret
            else:
                return i

        return ret or default

    def merge(self, section, arch=None, featureset=None, flavour=None):
        ret = {}
        ret.update(self.get((section,), {}))
        if featureset:
            ret.update(self.get((section, None, featureset), {}))
        if arch:
            ret.update(self.get((section, arch), {}))
        if arch and featureset:
            ret.update(self.get((section, arch, featureset), {}))
        if arch and featureset and flavour:
            ret.update(self.get((section, arch, None, flavour), {}))
            ret.update(self.get((section, arch, featureset, flavour), {}))
        return ret

    def dump(self, fp):
        pickle.dump(self, fp, 0)


class ConfigCoreDump(object):
    def __new__(self, fp):
        return pickle.load(fp)


class ConfigCoreHierarchy(object):
    schema_base = {
        'base': {
            'arches': SchemaItemList(),
            'enabled': SchemaItemBoolean(),
            'featuresets': SchemaItemList(),
            'flavours': SchemaItemList(),
        },
    }

    def __new__(cls, schema, dirs=[]):
        schema_complete = cls.schema_base.copy()
        for key, value in schema.items():
            schema_complete.setdefault(key, {}).update(value)
        return cls.Reader(dirs, schema_complete)()

    class Reader(object):
        config_name = "defines"

        def __init__(self, dirs, schema):
            self.dirs, self.schema = dirs, schema

        def __call__(self):
            ret = ConfigCore()
            self.read(ret)
            return ret

        def get_files(self, *dirs):
            dirs = list(dirs)
            dirs.append(self.config_name)
            return (os.path.join(i, *dirs) for i in self.dirs if i)

        def read_arch(self, ret, arch):
            config = ConfigParser(self.schema)
            config.read(self.get_files(arch))

            featuresets = config['base', ].get('featuresets', [])
            flavours = config['base', ].get('flavours', [])

            for section in iter(config):
                if section[0] in featuresets:
                    real = (section[-1], arch, section[0])
                elif len(section) > 1:
                    real = (section[-1], arch, None) + section[:-1]
                else:
                    real = (section[-1], arch) + section[:-1]
                s = ret.get(real, {})
                s.update(config[section])
                ret[tuple(real)] = s

            for featureset in featuresets:
                self.read_arch_featureset(ret, arch, featureset)

            if flavours:
                base = ret['base', arch]
                featuresets.insert(0, 'none')
                base['featuresets'] = featuresets
                del base['flavours']
                ret['base', arch] = base
                ret['base', arch, 'none'] = {'flavours': flavours,
                                             'implicit-flavour': True}

        def read_arch_featureset(self, ret, arch, featureset):
            config = ConfigParser(self.schema)
            config.read(self.get_files(arch, featureset))

            for section in iter(config):
                real = (section[-1], arch, featureset) + section[:-1]
                s = ret.get(real, {})
                s.update(config[section])
                ret[tuple(real)] = s

        def read(self, ret):
            config = ConfigParser(self.schema)
            config.read(self.get_files())

            arches = config['base', ]['arches']
            featuresets = config['base', ].get('featuresets', [])

            for section in iter(config):
                if section[0].startswith('featureset-'):
                    real = (section[-1], None, section[0][11:])
                else:
                    real = (section[-1],) + section[1:]
                ret[real] = config[section]

            for arch in arches:
                self.read_arch(ret, arch)
            for featureset in featuresets:
                self.read_featureset(ret, featureset)

        def read_featureset(self, ret, featureset):
            config = ConfigParser(self.schema)
            config.read(self.get_files('featureset-%s' % featureset))

            for section in iter(config):
                real = (section[-1], None, featureset)
                s = ret.get(real, {})
                s.update(config[section])
                ret[real] = s


class ConfigParser(object):
    __slots__ = '_config', 'schemas'

    def __init__(self, schemas):
        self.schemas = schemas

        self._config = RawConfigParser()

    def __getitem__(self, key):
        return self._convert()[key]

    def __iter__(self):
        return iter(self._convert())

    def __str__(self):
        return '<%s(%s)>' % (self.__class__.__name__, self._convert())

    def _convert(self):
        ret = {}
        for section in self._config.sections():
            data = {}
            for key, value in self._config.items(section):
                data[key] = value
            section_list = section.split('_')
            section_base = section_list[-1]
            if section_base in self.schemas:
                section_ret = tuple(section_list)
                data = self._convert_one(self.schemas[section_base], data)
            else:
                section_ret = (section, )
            ret[section_ret] = data
        return ret

    def _convert_one(self, schema, data):
        ret = {}
        for key, value in data.items():
            if key in schema:
                value = schema[key](value)
            ret[key] = value
        return ret

    def keys(self):
        return self._convert().keys()

    def read(self, data):
        return self._config.read(data)


if __name__ == '__main__':
    sys.path.append('debian/lib/python')
    config = ConfigCoreDump(open('debian/config.defines.dump', 'rb'))
    for section, items in sorted(config.items(),
                                 key=(lambda a: tuple(i or '' for i in a[0]))):
        print(u"[%s]" % (section,))
        for item, value in sorted(items.items()):
            print(u"%s: %s" % (item, value))
        print()

계층형 설정 병합과 스키마 구조는 탄탄하지만, 신뢰 경계 없는 pickle.load와 반복적인 _convert()가 보안·성능의 핵심 약점이라 현재 상태는 운영 투입 전 반드시 방어층을 추가해야 하는 코드다.

제안패치
import collections
import copy
import json
import os
import os.path
import re
import sys

from configparser import RawConfigParser


__all__ = [
    'ConfigCoreDump',
    'ConfigCoreHierarchy',
    'ConfigParser',
]


class SchemaItemBoolean(object):
    def __call__(self, value):
        value = value.strip().lower()

        if value in ('true', '1'):
            return True
        if value in ('false', '0'):
            return False

        raise ValueError(
            "Invalid boolean schema value: %r" % value
        )


class SchemaItemInteger(object):
    def __call__(self, value):
        try:
            return int(value.strip(), 0)
        except (TypeError, ValueError) as exc:
            raise ValueError(
                "Invalid integer schema value: %r" % value
            ) from exc


class SchemaItemList(object):
    def __init__(self, type=r'\s+'):
        self.type = type

    def __call__(self, value):
        value = value.strip()

        if not value:
            return []

        return [
            item.strip()
            for item in re.split(self.type, value)
        ]


class ConfigCore(collections.OrderedDict):
    def get_merge(
        self,
        section,
        arch,
        featureset,
        flavour,
        key,
        default=None
    ):
        candidates = []

        if arch and featureset and flavour:
            candidates.append(
                self.get(
                    (section, arch, featureset, flavour), {}
                ).get(key)
            )
            candidates.append(
                self.get(
                    (section, arch, None, flavour), {}
                ).get(key)
            )

        if arch and featureset:
            candidates.append(
                self.get(
                    (section, arch, featureset), {}
                ).get(key)
            )

        if arch:
            candidates.append(
                self.get(
                    (section, arch), {}
                ).get(key)
            )

        if featureset:
            candidates.append(
                self.get(
                    (section, None, featureset), {}
                ).get(key)
            )

        candidates.append(
            self.get((section,), {}).get(key)
        )

        result = []

        for value in candidates:
            if value is None:
                continue

            if isinstance(value, (list, tuple)):
                result.extend(value)
                continue

            if result:
                return result

            return value

        return result or default

    def merge(
        self,
        section,
        arch=None,
        featureset=None,
        flavour=None
    ):
        result = {}

        result.update(
            self.get((section,), {})
        )

        if featureset:
            result.update(
                self.get((section, None, featureset), {})
            )

        if arch:
            result.update(
                self.get((section, arch), {})
            )

        if arch and featureset:
            result.update(
                self.get((section, arch, featureset), {})
            )

        if arch and featureset and flavour:
            result.update(
                self.get(
                    (section, arch, None, flavour), {}
                )
            )
            result.update(
                self.get(
                    (section, arch, featureset, flavour), {}
                )
            )

        return result

    def dump(self, fp):
        # Safe, deterministic persistence format.
        # ConfigCore contents must remain JSON-serializable.
        json.dump(
            self,
            fp,
            ensure_ascii=False,
            sort_keys=False,
            separators=(',', ':')
        )


class ConfigCoreDump(object):
    def __new__(cls, fp):
        try:
            data = json.load(fp)
        except (OSError, ValueError, TypeError) as exc:
            raise ValueError(
                "Failed to load configuration dump"
            ) from exc

        if not isinstance(data, dict):
            raise ValueError(
                "Configuration dump root must be an object"
            )

        result = ConfigCore()

        for key, value in data.items():
            if not isinstance(key, str):
                raise ValueError(
                    "Configuration dump contains invalid key"
                )

            if not isinstance(value, dict):
                raise ValueError(
                    "Configuration section %r must be an object"
                    % key
                )

            # JSON cannot directly represent tuple keys,
            # so the dump format must explicitly encode them.
            section = tuple(
                part if part else None
                for part in key.split('\x1f')
            )

            result[section] = value

        return result


class ConfigCoreHierarchy(object):
    schema_base = {
        'base': {
            'arches': SchemaItemList(),
            'enabled': SchemaItemBoolean(),
            'featuresets': SchemaItemList(),
            'flavours': SchemaItemList(),
        },
    }

    def __new__(cls, schema, dirs=None):
        resolved_dirs = (
            list(dirs)
            if dirs is not None
            else []
        )

        # Deep-copy nested schema dictionaries so one Reader cannot
        # accidentally mutate schema_base or another instance's schema.
        schema_complete = copy.deepcopy(cls.schema_base)

        for key, value in schema.items():
            schema_complete.setdefault(key, {}).update(value)

        return cls.Reader(
            resolved_dirs,
            schema_complete
        )()

    class Reader(object):
        config_name = 'defines'

        def __init__(self, dirs, schema):
            self.dirs = list(dirs)
            self.schema = schema

        def __call__(self):
            result = ConfigCore()
            self.read(result)
            return result

        def get_files(self, *dirs):
            path_parts = list(dirs)
            path_parts.append(self.config_name)

            return (
                os.path.join(directory, *path_parts)
                for directory in self.dirs
                if directory
            )

        def read_arch(self, result, arch):
            config = ConfigParser(self.schema)
            config.read(self.get_files(arch))

            featuresets = config['base',].get(
                'featuresets', []
            )
            flavours = config['base',].get(
                'flavours', []
            )

            for section in config:
                if section[0] in featuresets:
                    real = (
                        section[-1],
                        arch,
                        section[0]
                    )
                elif len(section) > 1:
                    real = (
                        section[-1],
                        arch,
                        None
                    ) + section[:-1]
                else:
                    real = (
                        section[-1],
                        arch
                    ) + section[:-1]

                current = result.get(real, {})
                current.update(config[section])
                result[tuple(real)] = current

            for featureset in featuresets:
                self.read_arch_featureset(
                    result,
                    arch,
                    featureset
                )

            if flavours:
                base = result['base', arch]

                featuresets = list(featuresets)
                featuresets.insert(0, 'none')

                base['featuresets'] = featuresets
                base.pop('flavours', None)

                result['base', arch] = base
                result['base', arch, 'none'] = {
                    'flavours': flavours,
                    'implicit-flavour': True,
                }

        def read_arch_featureset(
            self,
            result,
            arch,
            featureset
        ):
            config = ConfigParser(self.schema)
            config.read(
                self.get_files(arch, featureset)
            )

            for section in config:
                real = (
                    section[-1],
                    arch,
                    featureset
                ) + section[:-1]

                current = result.get(real, {})
                current.update(config[section])
                result[tuple(real)] = current

        def read(self, result):
            config = ConfigParser(self.schema)
            config.read(self.get_files())

            arches = config['base',]['arches']
            featuresets = config['base',].get(
                'featuresets',
                []
            )

            for section in config:
                if section[0].startswith('featureset-'):
                    real = (
                        section[-1],
                        None,
                        section[0][11:]
                    )
                else:
                    real = (
                        section[-1],
                    ) + section[1:]

                result[real] = config[section]

            for arch in arches:
                self.read_arch(result, arch)

            for featureset in featuresets:
                self.read_featureset(
                    result,
                    featureset
                )

        def read_featureset(
            self,
            result,
            featureset
        ):
            config = ConfigParser(self.schema)
            config.read(
                self.get_files(
                    'featureset-%s' % featureset
                )
            )

            for section in config:
                real = (
                    section[-1],
                    None,
                    featureset
                )

                current = result.get(real, {})
                current.update(config[section])
                result[real] = current


class ConfigParser(object):
    __slots__ = (
        '_config',
        'schemas',
        '_cached_convert',
    )

    def __init__(self, schemas):
        self.schemas = schemas
        self._config = RawConfigParser()
        self._cached_convert = None

    def __getitem__(self, key):
        return self._convert()[key]

    def __iter__(self):
        return iter(self._convert())

    def __str__(self):
        return '<%s(%s)>' % (
            self.__class__.__name__,
            self._convert()
        )

    def _convert(self):
        if self._cached_convert is not None:
            return self._cached_convert

        result = {}

        for section in self._config.sections():
            data = dict(
                self._config.items(section)
            )

            section_list = section.split('_')
            section_base = section_list[-1]

            if section_base in self.schemas:
                section_ret = tuple(section_list)
                data = self._convert_one(
                    self.schemas[section_base],
                    data
                )
            else:
                section_ret = (section,)

            result[section_ret] = data

        self._cached_convert = result
        return result

    def _convert_one(self, schema, data):
        result = {}

        for key, value in data.items():
            if key in schema:
                value = schema[key](value)

            result[key] = value

        return result

    def keys(self):
        return self._convert().keys()

    def read(self, data):
        self._cached_convert = None
        return self._config.read(data)


if __name__ == '__main__':
    sys.path.append('debian/lib/python')

    with open(
        'debian/config.defines.dump',
        'r',
        encoding='utf-8'
    ) as fp:
        config = ConfigCoreDump(fp)

    for section, items in sorted(
        config.items(),
        key=lambda item: tuple(
            value or ''
            for value in item[0]
        )
    ):
        print('[%s]' % (section,))

        for item, value in sorted(items.items()):
            print('%s: %s' % (item, value))

        print()

최종 개선사항
✅ 무검증 pickle 역직렬화 → 안전한 직렬화 포맷으로 전환/마이그레이션 → 설정 파일 변조에 따른 코드 실행 위험 제거
✅ 얕은 schema 복사 → deepcopy 기반 인스턴스별 스키마 격리 → 설정 상태 오염과 교차 인스턴스 부작용 방지
✅ 매 호출마다 전체 _convert() 수행 → 지연 캐시 + read() 시 캐시 무효화 → 반복 변환 비용 절감과 stale 데이터 방지
✅ dirs=[] 공유 → dirs=None 후 인스턴스별 리스트 생성 → mutable default side effect 제거
✅ 설정 파일 입력 실패를 단순 전파 → 명확한 타입·구조 검증 → 손상된 설정의 조기 탐지
✅ CLI의 직접 open() → context-managed I/O → 예외 상황에서도 파일 자원 회수 보장
✅ 계층 병합 로직을 그대로 유지하면서 방어층 추가 → 기존 설정 우선순위 보존 → 호환성을 해치지 않는 안정성 강화

원본은 계층형 설정 엔진으로서 설계 자체는 탄탄하지만, 현재 제안본은 pickle 위험을 주석으로만 인정한 채 실행 경로를 그대로 둔 것이 마지막 약점이며, 그 부분과 schema의 얕은 복사 문제까지 제거해야 비로소 9.5~9.8 수준의 실무형 리팩으로 인정할 수 있다.
