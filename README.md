# OpenTrench: public works, out in the open

> CivicTech track · 24-Hour Web Development Hackathon · IIITDM Kancheepuram

**Live app:** `<add your hosted link here>`
**Demo video (2–5 min):** `<add link, or the file name if it is in this repo>`
**Team:** `<team name, members>`

OpenTrench puts every road, drain and pipeline project on one public map, warns departments before two crews dig the same street, and gives residents a simple way to follow up when work stalls or goes wrong.

---

## 1. The problem

(From the CivicTech problem statement: *Improving Transparency and Coordination in Public Works Projects.*)

Residents see roads dug up, drains under repair and pipelines being laid every day, but have no way to learn what is being done, who is responsible, or how long it will take. Departments and contractors often plan without knowing what others have scheduled, so the same road gets dug up several times in a few months. When work stalls or is done badly there is no simple way to follow up or hold anyone accountable.

| Problem in the brief | What OpenTrench does about it |
|---|---|
| Residents can't tell the purpose, timeline or responsible authority | Every project has a public page: purpose, department, contractor, budget, dates, progress bar and a progress log |
| Nobody explains why work is delayed | Delayed projects show a plain-language reason and the number of days overdue |
| The same road is dug up repeatedly | A clash engine flags projects that are close in space and time, and the planner checks new projects before they are published |
| No way to follow up or hold anyone accountable | Residents report issues (with an optional photo), back each other's reports, and the responsible department is emailed |
| Residents can't plan around disruption | Time machine slider, "Near me", and a Follow list show what is happening, where and when |
| Authorities miss early warnings from citizens | Reports are tied to a project and a department, with a public scoreboard showing who is on time |

## 2. Features

**For residents**
- **Public map** of every project: ongoing work shown as moving amber lines, late work as pulsing red pins, dashed red rings where projects clash.
- **Project panel** with purpose, department, contractor, budget, timeline, why it is late, clash warnings, progress log and the reports filed against it.
- **Time machine:** drag a slider (six months back to six months ahead) or press Play to watch which projects are active on any date.
- **Near me:** uses your location to show projects within 1.5 km, or tells you how far the nearest one is.
- **Search, status chips, department filter and sort** (needs attention first, name, budget, end date).
- **Follow** projects and filter to just the ones you follow. Following is stored in your browser.
- **"I am affected"** counter (one per person per project in your browser) so departments see how many people a project touches.
- **Report an issue** with type, description and an optional photo. You get a **confirmation email**, and the **responsible department is notified by email**.
- **Back a report** ("same here", one vote per report per browser), filter reports by type, sort by newest or most backed.
- **Share a project:** each project has a link such as `index.html#p=P-101` that opens its panel directly. The Share button copies it, or opens the phone share sheet.

**For departments**
- **Clash alerts:** projects within 150 m and under six months apart, with a plain suggestion ("co-schedule so the road is opened once").
- **Plan before you dig:** pick a spot and dates, see every clash within 150 m, then publish. **Find clash-free dates** searches the next year for the first window with no clash.
- **Accountability scoreboard:** money tracked, money sitting in delayed projects, average delay, estimated savings from co-scheduling, a department league table and a status mix chart. Click a department to filter the map.

**Everyone**
- Live ticker of late projects, clashes, live work and new reports; CSV download of all project data; keyboard shortcuts (`/` to search, `Esc` to close a panel); responsive layout; respects "reduce motion".

## 3. How the clash engine works

Defined in `pairClash()` in `index.html`:

1. Each project is a polyline of coordinates. Each segment is sampled at 10% steps so long or curved works are compared fairly.
2. The distance between two projects is the **minimum haversine distance** between any sampled points.
3. The time gap is the number of days between the end of the earlier project and the start of the later one. A delayed project is treated as running until at least 45 days from today, because its real end date is unknown.
4. Two projects **clash** when they are **within 150 m** and the gap is **at most 180 days** (a negative gap means the work overlaps).
5. The same function powers the clash cards, the red rings on the map, the planner's check and the "find clash-free dates" search.

The scoreboard's savings figure is an **estimate**, not a measurement: it assumes 15% of the smaller project's cost in each clash is re-excavation and road restoration that co-scheduling would avoid.

## 4. Tech stack and third-party services (disclosure)

No build step. One `index.html` plus a database and an email service.

| Used for | Library / service |
|---|---|
| Interactive maps | [Leaflet 1.9.4](https://leafletjs.com) (BSD-2) |
| Map tiles | Esri "Canvas / World Dark Gray" base and reference tiles, with OpenStreetMap contributors (attribution is shown on the map) |
| Shared database (published projects and reports) | [Supabase](https://supabase.com) REST API (PostgREST), publishable key, Row Level Security |
| Transactional email | [EmailJS](https://www.emailjs.com) browser SDK v4 |
| Fonts | Google Fonts: Big Shoulders Display, Figtree |

Everything else (animations, clash engine, charts, filters) is written from scratch in plain HTML, CSS and JavaScript. Project and report data shown in the demo is **simulated** (Chennai area).

## 5. Run it

### Locally
```bash
# any static server works; opening the file directly works for most features
python3 -m http.server 8000
# then open http://localhost:8000
```
Without keys, the map, clash engine, planner and all filters work. Published projects and reports then live only in that browser session, and emails are skipped with a note.

### Set up the shared database (Supabase)
The page reads and writes two tables over the REST API. In the Supabase SQL editor:

```sql
create table if not exists projects (
  id text primary key,
  data jsonb not null,
  created_at timestamptz not null default now()
);
create table if not exists reports (
  id text primary key,
  data jsonb not null,
  created_at timestamptz not null default now()
);

alter table projects enable row level security;
alter table reports  enable row level security;

create policy "public read projects"   on projects for select using (true);
create policy "public insert projects" on projects for insert with check (true);
create policy "public read reports"    on reports  for select using (true);
create policy "public insert reports"  on reports  for insert with check (true);
```
Then set `SB` (project URL + `/rest/v1`) and `SBKEY` (publishable key) near the top of the script section in `index.html`. Never put a `service_role` key in this file.

### Set up email (EmailJS)
1. Create an account, then **Email Services > Add New Service** (Gmail or Outlook). Copy the **Service ID**.
2. Copy your **Public Key** from **Account > General**.
3. Create two templates. In both, set **To Email** to `{{to_email}}`. In the department template also set **Reply To** to `{{reply_to}}`.
4. Fill in the `EJS` block in `index.html` (search for `email (EmailJS)`):

```js
const EJS={publicKey:'...',service:'service_...',tplResident:'template_...',tplDept:'template_...'};
const DEPT_FALLBACK='an inbox you control';          // receives any department alert with no address below
const DEPT_EMAIL={'Water Board':'', 'Telecom':'', ...}; // optional real department addresses
```
5. In EmailJS, add your hosted URL under **Allowed origins** so others can't reuse your key.

Variables the code sends to the templates:

| Variable | Meaning |
|---|---|
| `to_email` | Recipient (resident for the confirmation, department for the alert) |
| `reply_to`, `resident_email` | Resident's address (department alert only) |
| `report_id`, `date`, `category`, `message` | The report |
| `project_name`, `project_id`, `road`, `department`, `contractor` | The project |
| `has_photo` | "Yes" or "No" (photos are not attached) |
| `site_link` | Link back to the reports section |

Suggested template text is in `EMAIL_SETUP.md`. If the resident confirmation fails, the toast on screen shows EmailJS's error message for 9 seconds.

### Deploy
Any static host works (Netlify drop, Vercel, GitHub Pages, Cloudflare Pages). Upload `index.html`, then add the live URL to EmailJS **Allowed origins**.

## 6. Demo script (about 3 minutes)

1. **Hook (hero):** "One street. Three digs. Zero coordination." Point at the animation, then the live ticker.
2. **Map:** click the late culvert project (P-110) to show purpose, delay reason and days overdue. Turn on **Time machine** and press Play.
3. **Clash alerts:** open the Sardar Patel Rd clash. Three departments, one road, and the suggestion to share a trench.
4. **Planner:** press **Try a demo spot**, show the clash list, then **Find clash-free dates**, then publish and watch the confetti and the new pin.
5. **Resident report:** report an issue with your own email. Show the confirmation and department emails arriving.
6. **Scoreboard:** which department is late, how much money is stuck, and the estimated saving.
7. **Close:** share a project link (`#p=...`) and mention open data (CSV).

## 7. Privacy and security notes

- A resident's email is used only to send the two emails and is **never written to the database**. The department email shows it as the reply-to address so staff can respond.
- Resident-submitted text is escaped before it is shown, and photos over about 700 KB are not attached.
- The Supabase key in the file is the publishable (anonymous) key, so protection comes from the Row Level Security policies above.
- Follow, vote and "affected" choices are stored in `localStorage` (`ot_fol`, `ot_voted`, `ot_affd`).

## 8. Known limitations (honest list)

- Data is simulated and centred on Chennai. "Near me" tells you the nearest project if you are elsewhere.
- Anyone can publish a project and anyone can post a report: there is no sign-in or moderation. A real deployment needs department accounts, verified phone or email for reports, spam limits and an approval step.
- Vote counts, "I am affected" counts and follows are per browser and do not sync across users yet.
- Reports are linked to a project but not tracked through a status (acknowledged, resolved).
- Photos are stored inline in the report record and are not attached to emails.
- Email is sent from the browser through EmailJS, which has monthly limits on the free plan. A server-side function (Supabase Edge Function plus a mail API) is the production route.
- The savings figure on the scoreboard is an estimate based on a stated assumption.

## 9. Future work

Department login and role-based publishing; report status tracking with SMS or WhatsApp updates; import from real tender and work-order feeds; automatic contractor-performance history; map drawing for project routes (currently two points); multilingual interface (Tamil, Hindi); weather-aware delay prediction.

## 10. Mapping to the judging criteria

| Criterion | Where to look |
|---|---|
| Innovation (20%) | Clash engine plus planner that searches for clash-free dates; time-machine view; accountability scoreboard |
| Technical implementation (35%) | Geospatial clash engine (haversine, polyline sampling, date windows), shared database with RLS, email pipeline with error handling, deep links, accessible and responsive UI |
| Feasibility (20%) | Static front end, free-tier services, no hospital-style integrations; limitations and next steps listed above |
| Video presentation (15%) | See the demo script in section 6 |
| Deployment (10%) | Public static link, no setup needed for judges |

## 11. Files

```
index.html     the whole app (HTML, CSS, JavaScript)
README.md      this file
EMAIL_SETUP.md short guide and template text for EmailJS
demo video     add before submitting (the rules require it in the repo)
```

## 12. Credits

Leaflet, Esri and OpenStreetMap contributors, Supabase, EmailJS, Google Fonts. All project data and resident reports in the demo are simulated for the hackathon.
