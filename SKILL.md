---
name: youtube-collector
description: YouTube video search, comments, and transcript collection. No API key required. Searches by keyword (trending/relevance), collects best comments with batch delay, extracts subtitles. Triggers on "유튜브 수집", "유튜브 검색", "유튜브 댓글", "유튜브 자막", "youtube", "영상 수집".
---

# 유튜브 수집기 (YouTube Collector)

키워드 기반 유튜브 영상 검색 → 댓글 수집 → 자막 추출 → CSV + TXT 저장 → (선택) 분석까지 자동 수행합니다.
**API 키 없이** 바로 사용 가능합니다.

## 역할

당신은 유튜브 트렌드 분석가입니다.
사용자의 관심 주제를 파악하고, 유튜브에서 관련 영상과 댓글을 수집한 뒤, 핵심 인사이트를 도출합니다.
사용자와 한국어로 대화합니다.

---

## 전체 흐름

```
Phase 0: SETUP (최초 1회)
  └── pip install 의존성 (API 키 불필요)

Phase 1: INTAKE (정보 수집)
  └── 사용자 요청에서 키워드/건수/댓글 여부 자연스럽게 파악

Phase 2: COLLECT (영상/댓글/자막 수집)
  ├── PROJECT_DIR 생성
  ├── youtube_collect.py 작성 (아래 코드)
  └── python3 youtube_collect.py 실행 → CSV + TXT 저장

Phase 3: ANALYZE (선택적 분석)
  └── CSV/TXT를 Read 도구로 읽고 Claude가 직접 분석
```

> **중요:** 매 작업 시작 시 고유 폴더를 생성합니다.
> ```bash
> PROJECT_DIR="$HOME/Downloads/youtube_$(date +%Y%m%d_%H%M)"
> mkdir -p "$PROJECT_DIR"
> ```

---

# Phase 0: SETUP (최초 1회)

API 키가 필요 없습니다. 의존성만 설치하면 됩니다.

```bash
pip install -q yt-dlp youtube-comment-downloader youtube-transcript-api rich langdetect 2>/dev/null
```

---

# Phase 1: INTAKE

사용자 요청을 자연스럽게 파악하여 파라미터로 변환합니다.

| 사용자 표현 | 명령 |
|------------|------|
| "케데헌 유튜브 검색해줘" | `python3 youtube_collect.py search "케데헌" 10` |
| "BTS 관련 영상 5건" | `python3 youtube_collect.py search "BTS" 5 -r` |
| "케데헌 영상 + 댓글 100개씩" | `python3 youtube_collect.py search "케데헌" 5 -c 100` |
| "이 영상 댓글 모아줘" | `python3 youtube_collect.py comments "URL" 50` |
| "이 영상 자막 추출해줘" | `python3 youtube_collect.py transcript "URL"` |

기본값: 10건, 이번 주 조회수순

---

# Phase 2: COLLECT

## 실행

`$PROJECT_DIR/youtube_collect.py`로 아래 코드를 저장하고 실행합니다.

```bash
cd "$PROJECT_DIR" && python3 youtube_collect.py search "키워드" 10
```

## youtube_collect.py

```python
#!/usr/bin/env python3
"""유튜브 수집기 - 영상 검색 + 댓글 + 자막 (API 키 불필요)"""

import argparse, csv, json, logging, os, random, sys, time
from dataclasses import dataclass, field
from datetime import datetime
from itertools import islice
from pathlib import Path
from urllib.parse import quote

import yt_dlp
from youtube_comment_downloader import YoutubeCommentDownloader
from langdetect import detect, LangDetectException
from rich.console import Console
from rich.panel import Panel
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn, MofNCompleteColumn

console = Console()

# ── 검색 ──

@dataclass
class VideoResult:
    title: str; url: str; video_id: str; channel: str = ""; views: int = 0
    likes: int = 0; comment_count: int = 0; published: str = ""; duration: str = ""
    description: str = ""; subscribers: int = 0; category: str = ""
    tags: list[str] = field(default_factory=list)

def _get_details(video_ids):
    details = {}
    opts = {'quiet': True, 'no_warnings': True, 'skip_download': True}
    for vid in video_ids:
        try:
            with yt_dlp.YoutubeDL(opts) as ydl:
                info = ydl.extract_info(f'https://www.youtube.com/watch?v={vid}', download=False)
                cats = info.get('categories') or []
                details[vid] = {'like_count': int(info.get('like_count',0) or 0),
                    'comment_count': int(info.get('comment_count',0) or 0),
                    'tags': info.get('tags') or [], 'upload_date': info.get('upload_date',''),
                    'subscribers': int(info.get('channel_follower_count',0) or 0),
                    'category': cats[0] if cats else ''}
            time.sleep(random.uniform(1, 2))
        except: details[vid] = {'like_count':0,'comment_count':0,'tags':[],'upload_date':'','subscribers':0,'category':''}
    return details

def search_youtube(query, max_results=10, trending=True):
    opts = {'quiet': True, 'no_warnings': True, 'skip_download': True, 'extract_flat': True}
    if trending:
        search_url = f'https://www.youtube.com/results?search_query={quote(query)}&sp=CAMSBAgEEAE%3D'
    else:
        search_url = f'ytsearch{max_results}:{query}'
    try:
        with yt_dlp.YoutubeDL(opts) as ydl:
            info = ydl.extract_info(search_url, download=False)
        results = []
        for item in (info.get('entries') or []):
            if not item: continue
            vid = item.get('id',''); dur = int(item.get('duration') or 0)
            results.append(VideoResult(title=item.get('title',''),
                url=item.get('url','') or f"https://www.youtube.com/watch?v={vid}",
                video_id=vid, channel=item.get('uploader','') or item.get('channel',''),
                views=int(item.get('view_count',0) or 0),
                published=item.get('upload_date','') or '',
                duration=f"{dur//60}:{dur%60:02d}" if dur else "",
                description=item.get('description','') or ''))
        results = results[:max_results]
        if results:
            details = _get_details([r.video_id for r in results])
            for r in results:
                d = details.get(r.video_id, {})
                r.likes=d.get('like_count',0); r.comment_count=d.get('comment_count',0)
                r.tags=d.get('tags',[]); r.subscribers=d.get('subscribers',0)
                r.category=d.get('category','')
                if not r.published and d.get('upload_date'): r.published=d['upload_date']
        return results
    except Exception as e:
        console.print(f"[red]검색 오류: {e}[/red]"); return []

# ── 댓글 ──

@dataclass
class Comment:
    author: str; text: str; likes: int = 0; published: str = ""; language: str = ""

def _detect_lang(text):
    if not text or len(text.strip()) < 3: return "unknown"
    try: return detect(text)
    except: return "unknown"

def get_comments(video_url, max_comments=100):
    try:
        raw = list(islice(YoutubeCommentDownloader().get_comments_from_url(video_url), max_comments))
        return [Comment(author=c.get('author',''), text=c.get('text',''),
                likes=c.get('votes',0) or 0, published=c.get('time',''),
                language=_detect_lang(c.get('text',''))) for c in raw]
    except: return []

# ── 자막 ──

def get_transcript(video_url):
    try:
        from youtube_transcript_api import YouTubeTranscriptApi
        from youtube_transcript_api._errors import TranscriptsDisabled, NoTranscriptFound
    except ImportError:
        return {'success': False, 'error': 'youtube-transcript-api 미설치'}
    vid = ""
    if "watch?v=" in video_url: vid = video_url.split("watch?v=")[1].split("&")[0]
    elif "youtu.be/" in video_url: vid = video_url.split("youtu.be/")[1].split("?")[0]
    try:
        ytt = YouTubeTranscriptApi()
        tl = ytt.list(vid); t = tl.find_transcript(["ko","en"]); f = t.fetch()
        text = " ".join(s.text for s in f)
        return {'success': True, 'text': text, 'language': t.language_code,
                'language_name': t.language, 'is_generated': t.is_generated, 'segment_count': len(f)}
    except NoTranscriptFound: return {'success': False, 'error': '자막 없음'}
    except TranscriptsDisabled: return {'success': False, 'error': '자막 비활성화'}
    except Exception as e: return {'success': False, 'error': str(e)[:200]}

# ── 유틸 ──

def _fmt_date(d):
    if d and len(d)==8 and d.isdigit(): return f"{d[:4]}-{d[4:6]}-{d[6:]}"
    return d

def _downloads():
    win = Path("/mnt/c/Users") / os.getenv("USER","daniel") / "Downloads"
    linux = Path.home() / "Downloads"
    if win.is_dir(): return win
    if linux.is_dir(): return linux
    return Path.home()

# ── 메인 ──

def main():
    ap = argparse.ArgumentParser(description="유튜브 수집기 (API 키 불필요)")
    sub = ap.add_subparsers(dest="command")

    p_s = sub.add_parser("search")
    p_s.add_argument("query", nargs="+")
    p_s.add_argument("-n","--count", type=int, default=10)
    p_s.add_argument("-r", action="store_true", dest="relevance")
    p_s.add_argument("-c", type=int, default=0, dest="comments_count")

    p_c = sub.add_parser("comments")
    p_c.add_argument("url"); p_c.add_argument("count", nargs="?", type=int, default=50)

    p_t = sub.add_parser("transcript")
    p_t.add_argument("url")

    args = ap.parse_args()
    if not args.command: ap.print_help(); return

    dl = _downloads(); ts = datetime.now().strftime("%Y%m%d_%H%M")

    if args.command == "search":
        raw = args.query; count = args.count
        if len(raw)>1 and raw[-1].isdigit(): count=int(raw[-1]); raw=raw[:-1]
        queries = [q.strip() for q in " ".join(raw).split(",") if q.strip()]
        trending = not args.relevance; cmt_per = args.comments_count
        mode = "관련순" if args.relevance else "이번 주 조회수순"
        console.print(f"\n[bold blue]유튜브 검색[/bold blue] {', '.join(queries)} / {count}건 / {mode}")
        if cmt_per: console.print(f"  영상별 베스트 댓글 {cmt_per}개", style="dim")

        results, seen = [], set()
        for q in queries:
            for r in search_youtube(q, count, trending):
                if r.video_id not in seen: seen.add(r.video_id); results.append((q,r))
        if not results: console.print("[yellow]검색 결과 없음[/yellow]"); return
        console.print(f"\n  [green]총 {len(results)}건[/green]")

        console.print(Panel(f"[bold]검색어:[/bold] {', '.join(queries)}\n[bold]결과:[/bold] {len(results)}건",
            title="[bold blue]유튜브 검색 결과[/bold blue]", border_style="blue"))
        for i, (kw,r) in enumerate(results, 1):
            likes = f"  좋아요: {r.likes:,}" if r.likes else ""
            subs = f"  구독자: {r.subscribers:,}" if r.subscribers else ""
            console.print(f"\n  [bold]{i}. {r.title}[/bold]")
            console.print(f"     채널: {r.channel}{subs}  |  조회수: {r.views:,}{likes}")
            console.print(f"     {_fmt_date(r.published)}  |  {r.duration}  |  {r.category}")
            console.print(f"     [dim]{r.url}[/dim]")

        slug = queries[0].replace(" ","_").replace(",","_")[:20]
        # CSV
        cp = dl / f"youtube_{slug}_{ts}.csv"
        with open(cp,"w",encoding="utf-8-sig",newline="") as f:
            w=csv.writer(f)
            w.writerow(["검색어","제목","채널","구독자수","URL","조회수","좋아요","댓글수","길이","업로드날짜","카테고리","태그","영상ID","설명"])
            for kw,r in results:
                w.writerow([kw,r.title,r.channel,r.subscribers,r.url,r.views,r.likes,r.comment_count,
                    r.duration,_fmt_date(r.published),r.category,"|".join(r.tags[:10]),r.video_id,r.description])
        console.print(f"[green]CSV: {cp}[/green]")
        # TXT
        tp = dl / f"youtube_{slug}_{ts}.txt"
        lines = [f"유튜브 검색: {', '.join(queries)}", f"수집: {datetime.now().strftime('%Y-%m-%d %H:%M')}", f"총 {len(results)}건","="*70]
        for i,(kw,r) in enumerate(results,1):
            lines += [f"\n[{i}] {r.title}", f"    채널: {r.channel} (구독자 {r.subscribers:,})",
                f"    조회수: {r.views:,}  좋아요: {r.likes:,}  댓글: {r.comment_count:,}",
                f"    {_fmt_date(r.published)}  |  {r.duration}  |  {r.category}", f"    {r.url}", "-"*70]
        tp.write_text("\n".join(lines)+"\n", encoding="utf-8")
        console.print(f"[green]TXT: {tp}[/green]")

        # 댓글
        if cmt_per and results:
            console.print(f"\n  [cyan]{len(results)}개 영상 댓글 수집 중...[/cyan]")
            all_c = []
            with Progress(SpinnerColumn(),TextColumn("[progress.description]{task.description}"),
                    BarColumn(),MofNCompleteColumn(),console=console) as prog:
                task = prog.add_task("댓글", total=len(results))
                for i,(kw,r) in enumerate(results):
                    for c in get_comments(r.url, cmt_per):
                        all_c.append({"keyword":kw,"video_title":r.title,"channel":r.channel,
                            "views":r.views,"video_url":r.url,"author":c.author,"text":c.text,
                            "likes":c.likes,"time":c.published,"language":c.language})
                    prog.advance(task)
                    if i < len(results)-1: time.sleep(random.uniform(2,4))
            if all_c:
                ccp = dl / f"youtube_{slug}_{ts}_comments.csv"
                with open(ccp,"w",encoding="utf-8-sig",newline="") as f:
                    w=csv.writer(f)
                    w.writerow(["검색어","영상제목","채널","조회수","영상URL","작성자","댓글","좋아요","시기","언어"])
                    for c in all_c: w.writerow([c["keyword"],c["video_title"],c["channel"],c["views"],
                        c["video_url"],c["author"],c["text"],c["likes"],c["time"],c["language"]])
                console.print(f"[green]댓글 CSV: {ccp}[/green]")
                ctp = dl / f"youtube_{slug}_{ts}_comments.txt"
                lines2 = [f"유튜브 댓글: {', '.join(queries)}",f"총 {len(all_c)}건","="*70]
                cur=""
                for c in all_c:
                    if c["video_title"]!=cur:
                        cur=c["video_title"]; lines2+=[f"\n{'='*70}",f"영상: {cur}",f"채널: {c['channel']}  조회수: {c['views']:,}","-"*70]
                    lk=f" (♥ {c['likes']})" if c['likes'] else ""
                    lines2+=[f"\n  {c['author']}{lk}",f"  {c['text']}"]
                ctp.write_text("\n".join(lines2)+"\n",encoding="utf-8")
                console.print(f"[green]댓글 TXT: {ctp}[/green]")
                console.print(f"  댓글 {len(all_c)}건 수집 완료.")

    elif args.command == "comments":
        console.print(f"\n[bold blue]댓글 수집[/bold blue] (최대 {args.count}건)\n  {args.url}")
        comments = get_comments(args.url, args.count)
        if not comments: console.print("[yellow]댓글 없음[/yellow]"); return
        console.print(Panel(f"댓글: {len(comments)}건", title="[bold blue]유튜브 댓글[/bold blue]"))
        for i,c in enumerate(comments[:20],1):
            lk = f"  [red]♥ {c.likes}[/red]" if c.likes else ""
            console.print(f"\n  {i}. [bold]{c.author}[/bold]{lk}\n     {c.text[:200]}")
        vid = args.url.split("watch?v=")[1].split("&")[0] if "watch?v=" in args.url else "video"
        cp = dl/f"comments_{vid}_{ts}.csv"
        with open(cp,"w",encoding="utf-8-sig",newline="") as f:
            w=csv.writer(f); w.writerow(["영상URL","작성자","댓글","좋아요","시기","언어"])
            for c in comments: w.writerow([args.url,c.author,c.text,c.likes,c.published,c.language])
        console.print(f"[green]CSV: {cp}[/green]")

    elif args.command == "transcript":
        console.print(f"\n[bold blue]자막 추출[/bold blue]\n  {args.url}")
        sub = get_transcript(args.url)
        if not sub['success']: console.print(f"[yellow]{sub['error']}[/yellow]"); return
        console.print(f"  [green]성공[/green] ({sub['language_name']}, {sub['segment_count']}구간)")
        console.print(f"  [dim]{sub['text'][:200]}...[/dim]")
        # 영상 정보
        title=""; channel=""; views=0; likes=0; upload_date=""
        try:
            with yt_dlp.YoutubeDL({'quiet':True,'skip_download':True}) as ydl:
                info=ydl.extract_info(args.url,download=False)
                title=info.get('title',''); channel=info.get('uploader',''); views=int(info.get('view_count',0) or 0)
                likes=int(info.get('like_count',0) or 0); upload_date=info.get('upload_date','')
        except: pass
        safe = "".join(c for c in (title or "video").replace(" ","_") if c.isalnum() or c in "가-힣_-")[:30]
        cp = dl/f"youtube_자막_{safe}_{ts}.csv"
        with open(cp,"w",encoding="utf-8-sig",newline="") as f:
            w=csv.writer(f); w.writerow(["영상제목","채널","조회수","좋아요","업로드날짜","영상URL","언어","자동생성","구간수","전체자막"])
            w.writerow([title,channel,views,likes,upload_date,args.url,sub['language'],
                "Y" if sub['is_generated'] else "N", sub['segment_count'], sub['text']])
        console.print(f"[green]CSV: {cp}[/green]")
        tp = dl/f"youtube_자막_{safe}_{ts}.txt"
        tp.write_text("\n".join([f"영상: {title}",f"채널: {channel}",f"조회수: {views:,}  좋아요: {likes:,}",
            f"업로드: {_fmt_date(upload_date)}",f"URL: {args.url}",f"언어: {sub['language_name']} ({sub['language']})",
            f"자동생성: {'예' if sub['is_generated'] else '아니오'}",f"총 {sub['segment_count']}구간",
            "="*60,"",sub['text']]),encoding="utf-8")
        console.print(f"[green]TXT: {tp}[/green]")

if __name__ == "__main__": main()
```

---

# Phase 3: ANALYZE (선택적)

사용자가 "분석해줘", "요약해줘" 등을 요청하면 실행합니다.

1. Read 도구로 CSV 또는 TXT 파일 읽기
2. Claude가 직접 분석
3. 마크다운으로 보고

| 유형 | 사용자 표현 | Claude가 하는 일 |
|------|-----------|----------------|
| 요약 | "요약해줘" | 수집된 영상 핵심 내용 요약 |
| 트렌드 | "트렌드 알려줘" | 조회수/좋아요 분포, 채널별 분석 |
| 댓글 분석 | "댓글 분석해줘" | 감성 분류, 주요 의견, 언어별 분포 |
| 인사이트 | "인사이트 뽑아줘" | 핵심 발견 5개 + 시사점 |
| 종합 | "분석해줘" | 위 전부 간략 수행 |

---

# 에러 대응

| 에러 | 해결 |
|------|------|
| 모듈 미설치 | `pip install yt-dlp youtube-comment-downloader youtube-transcript-api rich langdetect` |
| 댓글 0건 | 해당 영상 댓글 비활성화 또는 일시 차단. 잠시 후 재시도. |
| 자막 없음 | 해당 영상에 자막(수동/자동생성)이 없는 경우. |
| 검색 결과 없음 | 키워드 변경 또는 `-r` 옵션으로 관련순 시도. |
