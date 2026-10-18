#!/usr/bin/env python3
"""Забирает новые посты канала Telegram и сохраняет их в blog/posts.json + blog/img/."""
import json, os, sys, urllib.request, urllib.parse, urllib.error

TOKEN = os.environ.get("TG_TOKEN", "").strip()
CHANNEL = os.environ.get("TG_CHANNEL", "").strip().lstrip("@").lower()
API = os.environ.get("TG_API", "https://api.telegram.org")
DATA, IMG, MAX_PHOTOS, MAX_POSTS = "blog/posts.json", "blog/img", 4, 200
if not TOKEN:
    sys.exit("Не задан секрет TG_TOKEN")

def call(method, **params):
    url = f"{API}/bot{TOKEN}/{method}?" + urllib.parse.urlencode(params)
    try:
        with urllib.request.urlopen(url, timeout=30) as r:
            d = json.load(r)
    except urllib.error.HTTPError as e:
        try:
            d = json.loads(e.read().decode())
        except Exception:
            d = {"ok": False, "description": str(e)}
    if not d.get("ok"):
        sys.exit("Ошибка Telegram API: %s" % d.get("description"))
    return d["result"]

def download(file_id, name):
    path = os.path.join(IMG, name)
    if not os.path.exists(path):
        info = call("getFile", file_id=file_id)
        os.makedirs(IMG, exist_ok=True)
        with urllib.request.urlopen(f"{API}/file/bot{TOKEN}/{info['file_path']}", timeout=60) as r, open(path, "wb") as f:
            f.write(r.read())
    return name

def from_our_channel(msg):
    if not CHANNEL:
        return True
    c = msg["chat"]
    return c.get("username", "").lower() == CHANNEL or str(c["id"]) == CHANNEL

data = {"offset": 0, "posts": []}
if os.path.exists(DATA):
    with open(DATA, encoding="utf-8") as f:
        data = json.load(f)
posts, offset = data.get("posts", []), data.get("offset", 0)

def find(mid, gid):
    for p in posts:
        if (gid and p.get("gid") == gid) or mid in p.get("mids", []):
            return p

def handle(msg, edited):
    mid, gid = msg["message_id"], msg.get("media_group_id")
    text = msg.get("text") or msg.get("caption") or ""
    photo = None
    if msg.get("photo"):
        ph = msg["photo"][-1]
        photo = (ph["file_id"], ph["file_unique_id"] + ".jpg")
    p = find(mid, gid)
    if p is None:
        if edited or (not text and not photo):
            return
        p = {"gid": gid, "mids": [], "date": msg["date"], "text": "", "photos": [], "link": None}
        posts.append(p)
    if mid not in p["mids"]:
        p["mids"].append(mid)
    p["date"] = min(p["date"], msg["date"])
    if text:
        p["text"] = text
    if photo and not edited and len(p["photos"]) < MAX_PHOTOS:
        name = download(*photo)
        if name not in p["photos"]:
            p["photos"].append(name)
    user = msg["chat"].get("username")
    if user:
        p["link"] = f"https://t.me/{user}/{min(p['mids'])}"

while True:
    updates = call("getUpdates", offset=offset, limit=100, timeout=0,
                   allowed_updates=json.dumps(["channel_post", "edited_channel_post"]))
    if not updates:
        break
    for u in updates:
        offset = u["update_id"] + 1
        msg = u.get("channel_post") or u.get("edited_channel_post")
        if msg and from_our_channel(msg):
            handle(msg, "edited_channel_post" in u)

posts.sort(key=lambda p: p["date"], reverse=True)
data = {"offset": offset, "posts": posts[:MAX_POSTS]}
os.makedirs("blog", exist_ok=True)
with open(DATA, "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=1)
print("Готово. Записей в блоге:", len(data["posts"]))
