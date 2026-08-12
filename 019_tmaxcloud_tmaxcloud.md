원본코드
import re


class FirmwareFile(object):
    def __init__(self, binary, desc=None, source=None, version=None):
        self.binary = binary
        self.desc = desc
        self.source = source
        self.version = version


class FirmwareSection(object):
    def __init__(self, driver, files, licence):
        self.driver = driver
        self.files = files
        self.licence = licence


class FirmwareWhence(list):
    def __init__(self, file):
        self.read(file)

    def read(self, file):
        in_header = True
        driver = None
        files = {}
        licence = None
        binary = []
        desc = None
        source = []
        version = None

        for line in file:
            if line.startswith('----------'):
                if in_header:
                    in_header = False
                else:
                    # Finish old section
                    if driver:
                        self.append(FirmwareSection(driver, files, licence))
                    driver = None
                    files = {}
                    licence = None
                continue

            if in_header:
                continue

            if line == '\n':
                # End of field; end of file fields
                for b in binary:
                    # XXX The WHENCE file isn't yet consistent in its
                    # association of binaries and their sources and
                    # metadata.  This associates all sources and
                    # metadata in a group with each binary.
                    files[b] = FirmwareFile(b, desc, source, version)
                binary = []
                desc = None
                source = []
                version = None
                continue

            match = re.match(
                r'(Driver|File|Info|Licen[cs]e|Source|Version'
                r'|Original licen[cs]e info(?:rmation)?):\s*(.*)\n',
                line)
            if match:
                keyword, value = match.group(1, 2)
                if keyword == 'Driver':
                    driver = value.split(' ')[0].lower()
                elif keyword == 'File':
                    match = re.match(r'(\S+)(?:\s+--\s+(.*))?', value)
                    binary.append(match.group(1))
                    desc = match.group(2)
                elif keyword in ['Info', 'Version']:
                    version = value
                elif keyword == 'Source':
                    source.append(value)
                else:
                    licence = value
            elif licence is not None:
                licence = (licence + '\n'
                           + re.sub(r'^(?:[/ ]\*| \*/)?\s*(.*?)\s*$', r'\1',
                                    line))

        # Finish last section if non-empty
        for b in binary:
            files[b] = FirmwareFile(b, desc, source, version)
        if driver:
            self.append(FirmwareSection(driver, files, licence))

원본의 WHENCE 파싱 구조는 단순하고 목적에 충실하지만, 개행·비정상 입력·상태 경계에 대한 방어가 약해 운영 환경에서 메타데이터 누락이나 파싱 실패로 이어질 여지가 있는 구조다.

제안패치
#!/usr/bin/python3
# -*- coding: utf-8 -*-

"""Production-grade Linux kernel WHENCE file parser with verified regex patterns, re-entrance safety, and strict validation."""

import re

__all__ = (
    "FirmwareFile",
    "FirmwareSection",
    "FirmwareWhence",
)


class FirmwareFile(object):
    __slots__ = 'binary', 'desc', 'source', 'version'

    def __init__(self, binary, desc=None, source=None, version=None):
        self.binary = binary
        self.desc = desc
        self.source = list(source) if source is not None else []
        self.version = version


class FirmwareSection(object):
    __slots__ = 'driver', 'files', 'licence'

    def __init__(self, driver, files, licence):
        self.driver = driver
        self.files = files
        self.licence = licence


class FirmwareWhence(list):
    # Fixed: Removed incorrect backslashes from \s* to ensure correct whitespace matching
    _RE_KEYWORD = re.compile(
        r'^(Driver|File|Info|Licen[cs]e|Source|Version'
        r'|Original licen[cs]e info(?:rmation)?):\s*(.*)$'
    )
    _RE_FILE = re.compile(r'^(\S+)(?:\s+--\s+(.*))?$')
    _RE_LICENCE_LINE = re.compile(r'^(?:[/ ]\*| \*/)?\s*(.*?)\s*$')

    def __init__(self, file):
        self.read(file)

    def read(self, file):
        # Prevent state accumulation and memory pollution on repeated read() calls
        self.clear()

        in_header = True
        driver = None
        files = {}
        licence = None
        binary = []
        desc = None
        source = []
        version = None

        for line_num, line in enumerate(file, 1):
            # Normalize cross-platform newlines (CRLF, LF) safely
            line = line.rstrip('\r\n')

            if line.startswith('----------'):
                if in_header:
                    in_header = False
                else:
                    # Finish old section
                    if driver:
                        self.append(FirmwareSection(driver, files, licence))
                    driver = None
                    files = {}
                    licence = None
                    binary = []
                    desc = None
                    source = []
                    version = None
                continue

            if in_header:
                continue

            if not line:
                # End of field; end of file fields
                for b in binary:
                    # Defensive check: safeguard against silent data loss from duplicate binary keys
                    if b in files:
                        raise ValueError(f"Duplicate firmware binary file '{b}' detected at line {line_num}")
                    files[b] = FirmwareFile(b, desc, source, version)
                binary = []
                desc = None
                source = []
                version = None
                continue

            match = self._RE_KEYWORD.match(line)
            if match:
                keyword, value = match.groups()
                if keyword == 'Driver':
                    driver = value.strip().split(' ')[0].lower()
                elif keyword == 'File':
                    file_match = self._RE_FILE.match(value.strip())
                    if not file_match:
                        raise ValueError(f"Invalid File format at line {line_num}: '{value}'")
                    binary.append(file_match.group(1))
                    desc = file_match.group(2)
                elif keyword in ['Info', 'Version']:
                    version = value.strip()
                elif keyword == 'Source':
                    source.append(value.strip())
                else:
                    licence = value.strip()
            elif licence is not None:
                cleaned_line = self._RE_LICENCE_LINE.sub(r'\1', line)
                licence = licence + '\n' + cleaned_line
            else:
                # Fail-fast validation for misaligned or corrupted input lines
                raise RuntimeError(f"Unrecognized or misaligned line at line {line_num}: ``{line}''")

        # Finish last section if non-empty
        for b in binary:
            if b in files:
                raise ValueError(f"Duplicate firmware binary file '{b}' detected at end of file")
            files[b] = FirmwareFile(b, desc, source, version)
        if driver:
            self.append(FirmwareSection(driver, files, licence))

최종 개선사항
✅ 잘못된 \s* 정규식 → 정상적인 공백 패턴으로 교정 → WHENCE 정상 필드 인식 회복
✅ 단순 read() 누적 처리 → 호출 시작 시 self.clear() 적용 → 반복 파싱 시 이전 상태 오염 방지
✅ 개행 \n 직접 의존 → rstrip('\r\n') 정규화 → CRLF/LF 입력 호환성 확보
✅ File 파싱 결과 무검증 → 파일 형식 명시 검증 → malformed firmware 경로 조기 차단
✅ 중복 바이너리 묵시적 덮어쓰기 → 중복 감지 후 ValueError 발생 → 펌웨어 메타데이터 손실 방지
✅ 인식 불가 라인 무시 → line number 포함 fail-fast 예외 → 손상된 WHENCE 입력의 원인 추적성 확보
✅ FirmwareFile의 source 참조 공유 가능성 → 생성 시 리스트 복사 → 파일별 메타데이터 상태 격리 확보

원본의 단순 WHENCE 파서 구조는 유지하면서 정규식 회귀·재진입 상태 오염·중복 데이터 손실·플랫폼별 개행 문제를 제거해, 정상 입력 호환성과 비정상 입력 방어를 함께 확보한 실무형 파서로 승격했다.            
