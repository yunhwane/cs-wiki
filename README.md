# cs-wiki

CS 학습 내용 정리 위키. MkDocs Material 기반.

## 로컬 실행

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

http://127.0.0.1:8000

## 빌드

```bash
mkdocs build
```

## 배포

`main` 푸시 시 GitHub Actions 가 `gh-pages` 브랜치로 자동 배포.
저장소 Settings → Pages → Source: `gh-pages` 설정 필요.

## 구조

```
docs/
├── index.md
├── os/              운영체제
├── network/         네트워크
├── database/        데이터베이스
├── data-structure/  자료구조
├── algorithm/       알고리즘
├── system-design/   시스템 디자인
├── language/        언어
├── security/        보안
└── git/             Git
```

## 새 문서 추가

1. `docs/<카테고리>/<주제>.md` 생성
2. `mkdocs.yml` `nav` 항목에 추가
