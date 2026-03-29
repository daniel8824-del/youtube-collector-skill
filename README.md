# YouTube Collector Skill

유튜브 영상 검색 + 댓글 + 자막 수집 Claude Code 스킬입니다. **API 키 없이** 바로 사용 가능합니다.

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.com/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## Features

- 키워드만 말하면 **유튜브 영상 자동 검색** (조회수순/관련순, 멀티 키워드)
- **베스트 댓글** 배치 수집 (영상별 N개, 자동 대기)
- **자막 추출** (한국어 우선 → 영어 폴백)
- 영상 상세 정보: 제목, 채널, 구독자수, 조회수, 좋아요, 댓글수, 카테고리, 태그
- 최종 결과물: **CSV + TXT**
- **API 키 불필요** — 설치 즉시 사용

## 사용 방법

### 1. Claude Code 스킬

```bash
git clone https://github.com/daniel8824-del/youtube-collector-skill.git ~/.claude/skills/youtube-collector
```

Claude Code에서:
```
"케데헌 유튜브 검색해줘"
"BTS 영상 5건 + 댓글 100개씩"
"이 영상 자막 추출해줘"
```

### 2. Google Antigravity

```bash
git clone https://github.com/daniel8824-del/youtube-collector-skill.git
cd youtube-collector-skill
```

`.agent/` 폴더가 자동 인식됩니다.

## 기술 스택

- `yt-dlp` — 영상 검색 + 상세 정보 (API 키 불필요)
- `youtube-comment-downloader` — 베스트 댓글 추출 (API 키 불필요)
- `youtube-transcript-api` — 자막 추출 (API 키 불필요)
- `langdetect` — 댓글 언어 감지

## License

MIT
