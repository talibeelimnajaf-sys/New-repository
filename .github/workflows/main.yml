import json, os, urllib.parse, urllib.request
from datetime import datetime, timezone

KEY = os.environ.get("YT_API_KEY", "")
CID = os.environ.get("CHANNEL_ID", "UC7pyV8zRn59yiHd1VEXhAcg")
STATE = "state.json"
DIGEST = os.environ.get("SCHEDULE", "") == "30 15 * * *" or os.environ.get("FORCE_DIGEST") == "1"
VIEW_JUMP = 300   # اتنے ویوز ایک چکر میں بڑھیں تو الرٹ
SUB_JUMP = 3      # سبسکرائبرز میں اتنا اضافہ تو الرٹ (کمی میں 2 بھی)


def yt(path, **p):
    p["key"] = KEY
    url = "https://www.googleapis.com/youtube/v3/" + path + "?" + urllib.parse.urlencode(p)
    with urllib.request.urlopen(url, timeout=30) as r:
        return json.load(r)


def snapshot():
    ch = yt("channels", part="snippet,statistics,contentDetails", id=CID)["items"][0]
    up = ch["contentDetails"]["relatedPlaylists"]["uploads"]
    items = yt("playlistItems", part="contentDetails", playlistId=up, maxResults=15)["items"]
    ids = ",".join(i["contentDetails"]["videoId"] for i in items)
    vids = yt("videos", part="snippet,statistics", id=ids)["items"] if ids else []
    s = ch["statistics"]
    return {
        "t": datetime.now(timezone.utc).isoformat(timespec="minutes"),
        "name": ch["snippet"]["title"],
        "subs": int(s.get("subscriberCount", 0)),
        "views": int(s.get("viewCount", 0)),
        "count": int(s.get("videoCount", 0)),
        "videos": {
            v["id"]: {
                "title": v["snippet"]["title"],
                "views": int(v["statistics"].get("viewCount", 0)),
                "likes": int(v["statistics"].get("likeCount", 0)),
                "comments": int(v["statistics"].get("commentCount", 0)),
            }
            for v in vids
        },
    }


def compare(old, new):
    a = []
    for vid, v in new["videos"].items():
        o = old["videos"].get(vid)
        if o is None:
            a.append(f"🆕 نئی ویڈیو: {v['title']} (https://youtu.be/{vid})")
            continue
        d = v["views"] - o["views"]
        if d >= VIEW_JUMP:
            a.append(f"🚀 ویوز میں تیزی: {v['title']} — {o['views']:,} سے {v['views']:,} (+{d:,})")
        if v["comments"] > o["comments"]:
            a.append(f"💬 نیا کمنٹ: {v['title']} (+{v['comments'] - o['comments']})")
    if new["count"] < old["count"]:
        a.append(f"⚠️ {old['count'] - new['count']} ویڈیو ہٹ گئی یا پرائیویٹ ہو گئی")
    ds = new["subs"] - old["subs"]
    if ds >= SUB_JUMP or ds <= -2:
        a.append(f"👥 سبسکرائبرز: {old['subs']} سے {new['subs']} ({ds:+d})")
    return a


def delta(a, b):
    d = a - b
    return f"{a:,} ({d:+,})"


def issue(title, body):
    tok, repo = os.environ.get("GITHUB_TOKEN"), os.environ.get("GITHUB_REPOSITORY")
    if not tok or not repo:
        print(title)
        print(body)
        return
    owner = repo.split("/")[0]
    data = json.dumps({"title": title, "body": f"@{owner}\n\n{body}"}).encode()
    req = urllib.request.Request(
        f"https://api.github.com/repos/{repo}/issues",
        data=data,
        method="POST",
        headers={
            "Authorization": f"Bearer {tok}",
            "Accept": "application/vnd.github+json",
            "Content-Type": "application/json",
        },
    )
    urllib.request.urlopen(req, timeout=30).read()


def main():
    new = snapshot()
    st = json.load(open(STATE, encoding="utf-8")) if os.path.exists(STATE) else {}
    old = st.get("last")
    alerts = compare(old, new) if old else []
    if alerts:
        issue("🔔 چینل الرٹ " + new["t"], "\n".join(f"- {x}" for x in alerts))
    if DIGEST:
        base = st.get("digest_base") or old or new
        top = sorted(new["videos"].items(), key=lambda kv: -kv[1]["views"])[:3]
        lines = [
            f"چینل: {new['name']}",
            f"- سبسکرائبرز: {delta(new['subs'], base['subs'])}",
            f"- کل ویوز: {delta(new['views'], base['views'])}",
            f"- ویڈیوز: {delta(new['count'], base['count'])}",
            "",
            "حالیہ ویڈیوز میں سب سے آگے:",
        ]
        for vid, v in top:
            lines.append(f"- {v['title']}: 👁 {v['views']:,} · 👍 {v['likes']:,} · 💬 {v['comments']:,} (https://youtu.be/{vid})")
        lines.append("")
        lines.append("تبدیلی پچھلی روزانہ رپورٹ کے مقابلے میں ہے۔")
        issue("📊 روزانہ رپورٹ " + new["t"][:10], "\n".join(lines))
        st["digest_base"] = new
    st["last"] = new
    with open(STATE, "w", encoding="utf-8") as f:
        json.dump(st, f, ensure_ascii=False, indent=1)


if __name__ == "__main__":
    main()
