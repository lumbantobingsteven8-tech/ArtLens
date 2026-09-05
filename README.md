import os
import sqlite3

from flask import Flask, flash, redirect, render_template, request, session, url_for
from werkzeug.security import check_password_hash, generate_password_hash

from helpers import login_required

app = Flask(__name__)

# A real deployment should read this from an environment variable instead.
app.secret_key = os.environ.get("ARTLENS_SECRET_KEY", "dev-secret-key-change-me")

DB_PATH = os.path.join(os.path.dirname(__file__), "art.db")


def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn


@app.after_request
def add_header(response):
    """Ensure responses aren't cached, so login/logout state stays fresh."""
    response.headers["Cache-Control"] = "no-cache, no-store, must-revalidate"
    return response


# ---------- Browse ----------

@app.route("/")
def index():
    db = get_db()

    technique = request.args.get("technique", "").strip()
    era = request.args.get("era", "").strip()

    query = "SELECT * FROM paintings WHERE 1=1"
    params = []
    if technique:
        query += " AND technique = ?"
        params.append(technique)
    if era:
        query += " AND era = ?"
        params.append(era)
    query += " ORDER BY artist, year"

    paintings = db.execute(query, params).fetchall()
    techniques = [r["technique"] for r in db.execute(
        "SELECT DISTINCT technique FROM paintings ORDER BY technique")]
    eras = [r["era"] for r in db.execute(
        "SELECT DISTINCT era FROM paintings ORDER BY era")]

    favorite_ids = set()
    if session.get("user_id"):
        rows = db.execute(
            "SELECT painting_id FROM favorites WHERE user_id = ?",
            (session["user_id"],),
        ).fetchall()
        favorite_ids = {r["painting_id"] for r in rows}

    db.close()
    return render_template(
        "index.html",
        paintings=paintings,
        techniques=techniques,
        eras=eras,
        selected_technique=technique,
        selected_era=era,
        favorite_ids=favorite_ids,
    )


@app.route("/painting/<int:painting_id>")
def painting(painting_id):
    db = get_db()
    p = db.execute("SELECT * FROM paintings WHERE id = ?", (painting_id,)).fetchone()
    if p is None:
        db.close()
        flash("That painting doesn't exist.")
        return redirect(url_for("index"))

    is_favorite = False
    if session.get("user_id"):
        row = db.execute(
            "SELECT 1 FROM favorites WHERE user_id = ? AND painting_id = ?",
            (session["user_id"], painting_id),
        ).fetchone()
        is_favorite = row is not None

    db.close()
    return render_template("painting.html", p=p, is_favorite=is_favorite)


# ---------- Compare ----------

@app.route("/compare")
def compare():
    db = get_db()
    all_paintings = db.execute("SELECT id, title, artist FROM paintings ORDER BY artist, title").fetchall()

    a_id = request.args.get("a", type=int)
    b_id = request.args.get("b", type=int)

    a = db.execute("SELECT * FROM paintings WHERE id = ?", (a_id,)).fetchone() if a_id else None
    b = db.execute("SELECT * FROM paintings WHERE id = ?", (b_id,)).fetchone() if b_id else None

    db.close()
    return render_template("compare.html", all_paintings=all_paintings, a=a, b=b)


# ---------- Favorites ----------

@app.route("/favorite/<int:painting_id>", methods=["POST"])
@login_required
def toggle_favorite(painting_id):
    db = get_db()
    existing = db.execute(
        "SELECT 1 FROM favorites WHERE user_id = ? AND painting_id = ?",
        (session["user_id"], painting_id),
    ).fetchone()

    if existing:
        db.execute(
            "DELETE FROM favorites WHERE user_id = ? AND painting_id = ?",
            (session["user_id"], painting_id),
        )
    else:
        db.execute(
            "INSERT OR IGNORE INTO favorites (user_id, painting_id) VALUES (?, ?)",
            (session["user_id"], painting_id),
        )
    db.commit()
    db.close()

    return redirect(request.referrer or url_for("index"))


@app.route("/favorites")
@login_required
def favorites():
    db = get_db()
    rows = db.execute(
        """SELECT paintings.* FROM paintings
           JOIN favorites ON favorites.painting_id = paintings.id
           WHERE favorites.user_id = ?
           ORDER BY paintings.artist""",
        (session["user_id"],),
    ).fetchall()
    db.close()
    return render_template("favorites.html", paintings=rows)


# ---------- Auth ----------

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "GET":
        return render_template("register.html")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")
    confirmation = request.form.get("confirmation", "")

    if not username or not password:
        flash("Username and password are required.")
        return redirect(url_for("register"))
    if password != confirmation:
        flash("Passwords don't match.")
        return redirect(url_for("register"))

    db = get_db()
    try:
        db.execute(
            "INSERT INTO users (username, hash) VALUES (?, ?)",
            (username, generate_password_hash(password)),
        )
        db.commit()
    except sqlite3.IntegrityError:
        flash("That username is already taken.")
        db.close()
        return redirect(url_for("register"))

    user = db.execute("SELECT id FROM users WHERE username = ?", (username,)).fetchone()
    db.close()

    session["user_id"] = user["id"]
    session["username"] = username
    flash("Welcome to ArtLens!")
    return redirect(url_for("index"))


@app.route("/login", methods=["GET", "POST"])
def login():
    session.clear()

    if request.method == "GET":
        return render_template("login.html")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")

    db = get_db()
    user = db.execute("SELECT * FROM users WHERE username = ?", (username,)).fetchone()
    db.close()

    if user is None or not check_password_hash(user["hash"], password):
        flash("Invalid username and/or password.")
        return redirect(url_for("login"))

    session["user_id"] = user["id"]
    session["username"] = user["username"]
    return redirect(url_for("index"))


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("index"))


if __name__ == "__main__":
    app.run(debug=True)
import os
import sqlite3

from flask import Flask, flash, redirect, render_template, request, session, url_for
from werkzeug.security import check_password_hash, generate_password_hash

from helpers import login_required

app = Flask(__name__)

# A real deployment should read this from an environment variable instead.
app.secret_key = os.environ.get("ARTLENS_SECRET_KEY", "dev-secret-key-change-me")

DB_PATH = os.path.join(os.path.dirname(__file__), "art.db")


def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn


@app.after_request
def add_header(response):
    """Ensure responses aren't cached, so login/logout state stays fresh."""
    response.headers["Cache-Control"] = "no-cache, no-store, must-revalidate"
    return response


# ---------- Browse ----------

@app.route("/")
def index():
    db = get_db()

    technique = request.args.get("technique", "").strip()
    era = request.args.get("era", "").strip()

    query = "SELECT * FROM paintings WHERE 1=1"
    params = []
    if technique:
        query += " AND technique = ?"
        params.append(technique)
    if era:
        query += " AND era = ?"
        params.append(era)
    query += " ORDER BY artist, year"

    paintings = db.execute(query, params).fetchall()
    techniques = [r["technique"] for r in db.execute(
        "SELECT DISTINCT technique FROM paintings ORDER BY technique")]
    eras = [r["era"] for r in db.execute(
        "SELECT DISTINCT era FROM paintings ORDER BY era")]

    favorite_ids = set()
    if session.get("user_id"):
        rows = db.execute(
            "SELECT painting_id FROM favorites WHERE user_id = ?",
            (session["user_id"],),
        ).fetchall()
        favorite_ids = {r["painting_id"] for r in rows}

    db.close()
    return render_template(
        "index.html",
        paintings=paintings,
        techniques=techniques,
        eras=eras,
        selected_technique=technique,
        selected_era=era,
        favorite_ids=favorite_ids,
    )


@app.route("/painting/<int:painting_id>")
def painting(painting_id):
    db = get_db()
    p = db.execute("SELECT * FROM paintings WHERE id = ?", (painting_id,)).fetchone()
    if p is None:
        db.close()
        flash("That painting doesn't exist.")
        return redirect(url_for("index"))

    is_favorite = False
    if session.get("user_id"):
        row = db.execute(
            "SELECT 1 FROM favorites WHERE user_id = ? AND painting_id = ?",
            (session["user_id"], painting_id),
        ).fetchone()
        is_favorite = row is not None

    db.close()
    return render_template("painting.html", p=p, is_favorite=is_favorite)


# ---------- Compare ----------

@app.route("/compare")
def compare():
    db = get_db()
    all_paintings = db.execute("SELECT id, title, artist FROM paintings ORDER BY artist, title").fetchall()

    a_id = request.args.get("a", type=int)
    b_id = request.args.get("b", type=int)

    a = db.execute("SELECT * FROM paintings WHERE id = ?", (a_id,)).fetchone() if a_id else None
    b = db.execute("SELECT * FROM paintings WHERE id = ?", (b_id,)).fetchone() if b_id else None

    db.close()
    return render_template("compare.html", all_paintings=all_paintings, a=a, b=b)


# ---------- Favorites ----------

@app.route("/favorite/<int:painting_id>", methods=["POST"])
@login_required
def toggle_favorite(painting_id):
    db = get_db()
    existing = db.execute(
        "SELECT 1 FROM favorites WHERE user_id = ? AND painting_id = ?",
        (session["user_id"], painting_id),
    ).fetchone()

    if existing:
        db.execute(
            "DELETE FROM favorites WHERE user_id = ? AND painting_id = ?",
            (session["user_id"], painting_id),
        )
    else:
        db.execute(
            "INSERT OR IGNORE INTO favorites (user_id, painting_id) VALUES (?, ?)",
            (session["user_id"], painting_id),
        )
    db.commit()
    db.close()

    return redirect(request.referrer or url_for("index"))


@app.route("/favorites")
@login_required
def favorites():
    db = get_db()
    rows = db.execute(
        """SELECT paintings.* FROM paintings
           JOIN favorites ON favorites.painting_id = paintings.id
           WHERE favorites.user_id = ?
           ORDER BY paintings.artist""",
        (session["user_id"],),
    ).fetchall()
    db.close()
    return render_template("favorites.html", paintings=rows)


# ---------- Auth ----------

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "GET":
        return render_template("register.html")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")
    confirmation = request.form.get("confirmation", "")

    if not username or not password:
        flash("Username and password are required.")
        return redirect(url_for("register"))
    if password != confirmation:
        flash("Passwords don't match.")
        return redirect(url_for("register"))

    db = get_db()
    try:
        db.execute(
            "INSERT INTO users (username, hash) VALUES (?, ?)",
            (username, generate_password_hash(password)),
        )
        db.commit()
    except sqlite3.IntegrityError:
        flash("That username is already taken.")
        db.close()
        return redirect(url_for("register"))

    user = db.execute("SELECT id FROM users WHERE username = ?", (username,)).fetchone()
    db.close()

    session["user_id"] = user["id"]
    session["username"] = username
    flash("Welcome to ArtLens!")
    return redirect(url_for("index"))


@app.route("/login", methods=["GET", "POST"])
def login():
    session.clear()

    if request.method == "GET":
        return render_template("login.html")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")

    db = get_db()
    user = db.execute("SELECT * FROM users WHERE username = ?", (username,)).fetchone()
    db.close()

    if user is None or not check_password_hash(user["hash"], password):
        flash("Invalid username and/or password.")
        return redirect(url_for("login"))

    session["user_id"] = user["id"]
    session["username"] = user["username"]
    return redirect(url_for("index"))


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("index"))


if __name__ == "__main__":
    app.run(debug=True)
"""
init_db.py

Builds art.db from schema.sql and seeds it with sample painting data.
Also generates simple placeholder SVG images (one per painting) so the
app runs immediately without needing external image files. Each
placeholder's gradient direction/contrast reflects the lighting
technique it represents -- swap these out under static/images/ for
real (public-domain) reproductions whenever you like.

Run this once before starting the app:
    python3 init_db.py
"""

import os
import sqlite3

DB_PATH = "art.db"
IMAGE_DIR = os.path.join("static", "images")

# (title, artist, year, technique, era, light_source, description)
PAINTINGS = [
    (
        "The Calling of Saint Matthew", "Caravaggio", "1599–1600",
        "Tenebrism", "Baroque",
        "A single unseen beam from the upper right",
        "A shaft of light cuts diagonally across a dim tavern, landing on "
        "Matthew's face at the exact moment he is singled out. Everything "
        "outside that beam is swallowed in near-black shadow, so the eye "
        "has nowhere else to go -- the light itself performs the calling."
    ),
    (
        "The Night Watch", "Rembrandt", "1642",
        "Chiaroscuro", "Baroque",
        "Warm, uneven light from the left, as if just past a doorway",
        "Rembrandt lets light fall unevenly across a crowd of militia "
        "members, brightening a few key figures -- notably a small girl "
        "in gold -- while others recede into shadow. The uneven lighting "
        "turns a static group portrait into a frozen, dramatic moment."
    ),
    (
        "Girl with a Pearl Earring", "Johannes Vermeer", "c. 1665",
        "Chiaroscuro", "Baroque",
        "Soft, even light from an implied window, front-left",
        "The light here is gentle rather than dramatic, wrapping softly "
        "around the girl's face and catching the pearl as a single bright "
        "point. The dark, undefined background has no light at all, which "
        "makes the illuminated face feel almost like it is glowing on its own."
    ),
    (
        "Mona Lisa", "Leonardo da Vinci", "1503–1519",
        "Sfumato", "High Renaissance",
        "Diffuse, ambient light with no hard source",
        "Leonardo blurs every edge -- especially around the eyes and mouth "
        "-- so that light seems to dissolve into shadow rather than meet "
        "it at a line. That softness is exactly why her expression reads "
        "differently depending on where you look."
    ),
    (
        "Christ in the House of His Parents (study)", "Georges de La Tour", "c. 1640",
        "Tenebrism", "Baroque",
        "A single candle held by a child, inside the scene",
        "Rather than lighting the scene from outside, de La Tour places "
        "the light source inside the painting itself -- a candle whose "
        "glow defines every face around it. The darkness isn't empty; it's "
        "shaped entirely by how far that one flame reaches."
    ),
    (
        "Judith Slaying Holofernes", "Artemisia Gentileschi", "1614–1620",
        "Tenebrism", "Baroque",
        "Hard, raking light from above-left",
        "Stark, high-contrast lighting falls across the central struggle, "
        "picking out muscle, blood, and expression while the background "
        "disappears entirely into black. The harshness of the light "
        "matches the violence of the scene -- there is no soft place to look."
    ),
    (
        "Assumption of the Virgin", "Titian", "1516–1518",
        "Cangiante", "High Renaissance / Venetian",
        "Radiant, golden light from above, unifying the whole composition",
        "Titian shifts hue rather than just value to suggest luminosity -- "
        "reds warm toward orange, blues lift toward gold near the light "
        "source. The effect is a painting that seems lit from within its "
        "own colour, not just from an external source."
    ),
    (
        "The School of Athens", "Raphael", "1509–1511",
        "Unione", "High Renaissance",
        "Even, architectural daylight from multiple openings",
        "Light is spread smoothly and evenly across the whole scene, with "
        "soft, gradual transitions between light and shadow rather than "
        "sharp contrast. That evenness lets dozens of figures share the "
        "space calmly, without any one of them being singled out by light."
    ),
    (
        "The Milkmaid", "Johannes Vermeer", "c. 1658",
        "Unione", "Baroque",
        "Steady daylight from a window, upper-left",
        "A calm, consistent light fills the room from a single window, "
        "picking out texture -- bread crust, woven basket, pouring milk -- "
        "without ever becoming dramatic. The light's job here is patience, "
        "not spectacle."
    ),
    (
        "Las Meninas", "Diego Velázquez", "1656",
        "Chiaroscuro", "Baroque",
        "Layered light from a doorway, a window, and a mirror",
        "Velázquez uses several light sources of different strengths at "
        "once, including a mirror reflecting the king and queen. The "
        "layering of bright and dim areas creates real depth, guiding the "
        "eye from the foreground children all the way to the lit doorway "
        "at the back."
    ),
    (
        "The Anatomy Lesson of Dr. Nicolaes Tulp", "Rembrandt", "1632",
        "Chiaroscuro", "Baroque",
        "A single overhead light on the cadaver and surrounding faces",
        "The body at the center is the brightest element in the painting, "
        "with light falling off sharply toward the edges where the "
        "onlookers' faces catch just enough illumination to register "
        "their individual expressions of curiosity and unease."
    ),
    (
        "Bacchus and Ariadne", "Titian", "1520–1523",
        "Cangiante", "High Renaissance / Venetian",
        "Bright midday sun with strong local color shifts",
        "Instead of darkening shadows toward black, Titian shifts them "
        "toward adjacent hues -- a blue sky deepens toward violet, skin "
        "warms toward rose -- keeping the whole canvas vivid even in its "
        "darkest passages."
    ),
]

TECHNIQUE_COLORS = {
    "Tenebrism": ("#050505", "#f2c14e"),
    "Chiaroscuro": ("#1a1a1a", "#e8b04b"),
    "Sfumato": ("#3b3540", "#c9b8a8"),
    "Cangiante": ("#7a1f3d", "#f4d35e"),
    "Unione": ("#2b3a55", "#f0e6d2"),
}


def slugify(title: str) -> str:
    return (
        title.lower()
        .replace(" ", "_")
        .replace(",", "")
        .replace("(", "")
        .replace(")", "")
        .replace("'", "")
        .replace(".", "")
    )


def make_placeholder_svg(title: str, artist: str, technique: str, path: str) -> None:
    """Generate a simple gradient SVG standing in for the real artwork."""
    dark, light = TECHNIQUE_COLORS.get(technique, ("#222222", "#dddddd"))
    svg = f"""<svg xmlns="http://www.w3.org/2000/svg" width="640" height="480" viewBox="0 0 640 480">
  <defs>
    <radialGradient id="g" cx="30%" cy="25%" r="85%">
      <stop offset="0%" stop-color="{light}"/>
      <stop offset="55%" stop-color="{dark}"/>
      <stop offset="100%" stop-color="#000000"/>
    </radialGradient>
  </defs>
  <rect width="640" height="480" fill="url(#g)"/>
  <text x="320" y="420" text-anchor="middle" font-family="Georgia, serif"
        font-size="22" fill="{light}" opacity="0.9">{title}</text>
  <text x="320" y="450" text-anchor="middle" font-family="Georgia, serif"
        font-size="15" fill="{light}" opacity="0.7">{artist}</text>
</svg>"""
    with open(path, "w") as f:
        f.write(svg)


def main():
    os.makedirs(IMAGE_DIR, exist_ok=True)

    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)

    conn = sqlite3.connect(DB_PATH)
    with open("schema.sql") as f:
        conn.executescript(f.read())

    cur = conn.cursor()
    for title, artist, year, technique, era, light_source, description in PAINTINGS:
        image_file = f"images/{slugify(title)}.svg"
        make_placeholder_svg(title, artist, technique, os.path.join("static", image_file))
        cur.execute(
            """INSERT INTO paintings
               (title, artist, year, technique, era, image_file, light_source, description)
               VALUES (?, ?, ?, ?, ?, ?, ?, ?)""",
            (title, artist, year, technique, era, image_file, light_source, description),
        )

    conn.commit()
    conn.close()
    print(f"Seeded {len(PAINTINGS)} paintings into {DB_PATH}")
    print(f"Generated {len(PAINTINGS)} placeholder images in {IMAGE_DIR}/")


if __name__ == "__main__":
    main()
# ArtLens

#### Video Demo: <URL HERE>

#### Description:

ArtLens is a Flask web application that lets users explore classical paintings
through the specific lens of **light** — how each artist's handling of light
shapes what a viewer notices, feels, and understands about a scene.

Rather than presenting paintings as a flat gallery, ArtLens organizes the
collection around lighting **technique** (chiaroscuro, tenebrism, sfumato,
cangiante, unione) and **era**, and pairs every painting with a short written
breakdown of where its light comes from and why that choice matters to the
composition.

## Features

- **Browse & filter** — a responsive grid of paintings that can be filtered
  by lighting technique or historical era.
- **Painting detail pages** — the full image alongside its light source and
  a written explanation of how that light shapes the painting's focal point
  and mood.
- **Compare mode** — pick any two paintings to view side by side, useful for
  contrasting approaches such as Caravaggio's tenebrism against Vermeer's
  soft chiaroscuro.
- **User accounts** — register and log in to save paintings to a personal
  "Favorites" collection, backed by a SQLite database.

## Tech Stack

- **Backend:** Python (Flask)
- **Database:** SQLite (accessed directly via Python's `sqlite3` module)
- **Frontend:** Jinja2 templates, Bootstrap 5, custom CSS
- **Auth:** Werkzeug's password hashing, Flask's signed-cookie sessions

## Project Structure

| File / Folder       | Purpose                                                        |
|----------------------|-----------------------------------------------------------------|
| `app.py`             | All Flask routes (browse, detail, compare, favorites, auth)     |
| `helpers.py`         | `login_required` decorator used to protect routes               |
| `schema.sql`         | SQLite schema for `users`, `paintings`, and `favorites`          |
| `init_db.py`         | Builds `art.db` and seeds sample painting data + placeholder art |
| `templates/`         | Jinja2 HTML templates (layout, index, painting, compare, etc.)   |
| `static/styles.css`  | Custom styling on top of Bootstrap                               |
| `static/images/`     | Painting images (placeholders generated by `init_db.py`)         |
| `requirements.txt`   | Python dependencies                                              |

## Design Choices

- **Placeholder artwork:** `init_db.py` generates a simple gradient SVG for
  each painting so the app runs immediately without needing external image
  files or network access. The gradient direction and contrast for each
  placeholder loosely reflect its lighting technique (e.g., stark
  high-contrast gradients for tenebrism, soft even gradients for unione).
  Swap in real (public-domain) reproductions under `static/images/` at any
  time — the database only stores the file path.
- **Cookie-based sessions instead of `flask-session`:** Flask's built-in
  signed-cookie sessions are used instead of server-side filesystem
  sessions, keeping the project dependency-free and easy to run anywhere.
- **Direct `sqlite3` instead of an ORM:** keeps the SQL explicit and easy to
  follow for a project of this size.

## How to Run It Locally

```bash
pip install -r requirements.txt
python3 init_db.py   # creates art.db and seeds sample data (run once)
python3 app.py
```

Then open the URL Flask prints (typically `http://127.0.0.1:5000`) in your
browser.

## Author

- **Name:** Steven Lumban Tobing
- **GitHub username:** <YOUR lumbantobingsteven8-tech>
- **edX username:** <YOUR Steven Lumban Tobing>
- **City, Country:** <BERAU, INDONESIA>
- **Date recorded:** <DATE>
Flask==3.1.3
Werkzeug==3.1.3
body {
    background-color: #f4f1ea;
    color: #2b2b2b;
}

.navbar {
    background-color: #1b1b1b;
}

.navbar-brand {
    font-family: Georgia, 'Times New Roman', serif;
    font-size: 1.4rem;
}

h1, h4, .card-title {
    font-family: Georgia, 'Times New Roman', serif;
}

.painting-card {
    background-color: #ffffff;
    border: none;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.08);
    transition: transform 0.15s ease-in-out;
}

.painting-card:hover {
    transform: translateY(-3px);
}

.card-img-top {
    aspect-ratio: 4 / 3;
    object-fit: cover;
<img width="640" height="480" alt="assumption_of_the_virgin" src="https://github.com/user-attachments/assets/ee1e8aa3-23f3-465a-b36d-8470ef7f3012" /><img width="640" height="480" alt="the_school_of_athens" src="https://github.com/user-attachments/assets/06645af7-acf3-414f-9407-1d05f1cd7fbf" />
<img width="640" height="480" alt="the_night_watch" src="https://github.com/user-attachments/assets/3a1d2097-ab8e-4550-9c76-6be11d29f473" />
<img width="640" height="480" alt="the_milkmaid" src="https://github.com/user-attachments/assets/9abaf6bd-1e02-4f78-8bce-d4209526c5bd" />
<img width="640" height="480" alt="the_calling_of_saint_matthew" src="https://github.com/user-attachments/assets/c87d9afe-3310-429d-a135-87d7bc3782e3" />
<img width="640" height="480" alt="the_anatomy_lesson_of_dr_nicolaes_tulp" src="https://github.com/user-attachments/assets/2f732ea0-984c-475e-9b09-a8ca1be849f6" />
<img width="640" height="480" alt="mona_lisa" src="https://github.com/user-attachments/assets/f29ee7fb-fb9c-4353-93da-443d884aeaaa" />
<img width="640" height="480" alt="las_meninas" src="https://github.com/user-attachments/assets/c0a5d642-86a5-4c91-9c9f-573d1a0bbd91" />
<img width="640" height="480" alt="judith_slaying_holofernes" src="https://github.com/user-attachments/assets/4d8cf6cf-2bc0-4e74-8d0c-ee5c6f826cdb" />
<img width="640" height="480" alt="girl_with_a_pearl_earring" src="https://github.com/user-attachments/assets/a10c4efb-ab85-43ce-95b2-2b1e847958bf" />
<img width="640" height="480" alt="christ_in_the_house_of_his_parents_study" src="https://github.com/user-attachments/assets/43ac3e55-d492-45a8-866a-478ce3b171a9" />
<img width="640" height="480" alt="bacchus_and_ariadne" src="https://github.com/user-attachments/assets/d837a693-99be-40b2-a480-a3db5437d27a" />
{% extends "layout.html" %} {% block title %}My Favorites{% endblock %} {% block main %}
My Favorites

{% for p in paintings %}
 {{ p.title }}
{{ p.title }}
{{ p.artist }}, {{ p.year }}
{{ p.technique }}
â˜… Remove
{% else %}
You haven't favorited any paintings yet. Browse the collection.
{% endfor %}
{% endblock %}
{% extends "layout.html" %} {% block title %}Browse{% endblock %} {% block main %}
ArtLens

Explore classical paintings through how they use light.



{% if selected_technique or selected_era %}
Clear filters
{% endif %}
{% for p in paintings %}
 {{ p.title }}
{{ p.title }}

{{ p.artist }}, {{ p.year }}

{{ p.technique }} {{ p.era }}
View details {% if session.user_id %}
{{ 'â˜… Favorited' if p.id in favorite_ids else 'â˜† Favorite' }}
{% endif %}
{% else %}
No paintings match those filters.

{% endfor %}
{% endblock %}
🕯️ ArtLens
Browse
Compare
{% if session.user_id %}
My Favorites
Log Out ({{ session.username }})
{% else %}
Log In
Register
{% endif %}
{% with messages = get_flashed_messages() %} {% if messages %} {% for message in messages %}
{{ message }}
{% endfor %} {% endif %} {% endwith %} {% block main %}{% endblock %}
{% extends "layout.html" %} {% block title %}Log In{% endblock %} {% block main %}
Log In

Username  
Password  
Log In
Don't have an account? Register here.

{% endblock %}
{% extends "layout.html" %} {% block title %}{{ p.title }}{% endblock %} {% block main %} ← Back to browse
{{ p.title }}
{{ p.title }}

{{ p.artist }} · {{ p.year }}

{{ p.technique }} {{ p.era }}

Light source

{{ p.light_source }}

How light shapes this painting

{{ p.description }}

{% if session.user_id %}
{{ 'â˜… Remove from Favorites' if is_favorite else 'â˜† Add to Favorites' }}
{% endif %} Compare this painting
{% endblock %}
{% extends "layout.html" %} {% block title %}Register{% endblock %} {% block main %}
Register

Username  
Password  
Confirm Password  
Register
{% endblock %}
