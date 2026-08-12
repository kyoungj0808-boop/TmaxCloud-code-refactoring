원본코드
#!/usr/bin/python3
import re
import sys

for line in open(sys.argv[1]):
    match = re.search(r'getopt\(argc, argv, "([\w:]*?)"\)', line)
    if match:
        options = match.group(1)
        break
else:
    raise RuntimeError

print('#define GETOPT_OPTIONS "%s"' % options)

print('#define GETOPT_CASE', end=' ')
for c in options:
    if c == ':' or c == 'T':
        continue
    print("case '%c':" % c, end=' ')
print()

원본은 작동 자체는 단순·명확하지만 파일 수명 관리와 CLI 계약, getopt() 파싱 실패 원인에 대한 방어가 없어 입력 한 번만 어긋나도 빌드 단계가 즉시 중단되는 취약한 빌드 보조 스크립트다.

제안패치
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""Robust getopt macro generator for kernel module build system with strict syntactic guards."""

import argparse
import re
from pathlib import Path


def parse_getopt_options(source_file: Path) -> str:
    """Safely extracts and validates the getopt option string from C source code."""
    try:
        content = source_file.read_text(encoding='utf-8')
    except OSError as e:
        # Preserve original OSError context for accurate system-level diagnosis
        raise

    # [Defensive Design] Multi-line tolerant regex matching for getopt call
    match = re.search(r'getopt\s*\(\s*argc\s*,\s*argv\s*,\s*"([^"]+)"\s*\)', content, re.DOTALL)
    if not match:
        raise RuntimeError(f"Target getopt() pattern not found or malformed in source file: {source_file}")

    options = match.group(1)

    # [Integrity Guard] Validate option string syntax to prevent C macro/switch-case corruption.
    # Standard getopt options consist of alphanumeric characters, colons (:), and allowed special indicators.
    if not re.fullmatch(r'[a-zA-Z0-9:]+', options):
        raise RuntimeError(f"Invalid characters detected in getopt option string '{options}': must be alphanumeric and colons only.")

    return options


def generate_macro_output(options: str) -> None:
    """Generates strictly validated C preprocessor definitions and case clauses to stdout."""
    print(f'#define GETOPT_OPTIONS "{options}"')

    print('#define GETOPT_CASE', end=' ')
    for c in options:
        if c == ':' or c == 'T':
            continue
        print(f"case '{c}':", end=' ')
    print()


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description="Generate getopt macros from C source code with strict integrity checks.")
    parser.add_argument("source", type=Path, help="Path to the C source file containing getopt()")
    args = parser.parse_args()

    options_str = parse_getopt_options(args.source)
    generate_macro_output(options_str)

최종 개선사항
✅ sys.argv 직접 접근 → argparse 기반 CLI 계약 검증 → 인자 누락 시 예측 가능한 오류 처리
✅ 파일 직접 open() → Path.read_text() 기반 자동 자원 관리 → 파일 디스크립터 누수 방지
✅ 단일 라인 중심 정규식 → 공백·개행을 허용하는 getopt() 탐색 → 소스 형식 변화에 대한 내구성 강화
✅ 패턴 미검출 시 무정보 RuntimeError → 대상 파일과 원인을 포함한 명시적 예외 → 빌드 실패 원인 추적성 향상
✅ 옵션 문자열 무검증 → 허용 문자 검증 추가 → 생성되는 C 매크로·switch 구문의 무결성 강화
✅ 파싱과 출력 로직 혼합 → parse_getopt_options() / generate_macro_output() 분리 → 검증·테스트·유지보수성 향상

원본의 단순한 빌드 보조 스크립트라는 목적은 유지하면서 CLI·파일 처리·파싱 실패·출력 무결성의 방어선을 확보해, 과설계 없이 실무 빌드 환경에서 더 오래 버티는 구조로 승격되었다.
