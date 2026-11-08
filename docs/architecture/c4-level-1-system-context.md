# C4 Level 1: System context

Who uses Can't Run Alone and which external systems it depends on. Arrow convention: **A → B means A starts the interaction.**

![C4 Level 1 system context](images/c4-level-1-system-context.png)

```mermaid
flowchart TB
    visitor(["<b>Visitor</b><br/>[Person]<br/>Browses and registers"])
    runner(["<b>Runner</b><br/>[Person]<br/>Finds buddies, organizes<br/>and joins runs, rates partners"])
    moderator(["<b>Moderator</b><br/>[Person]<br/>Reviews reports and<br/>suspends accounts"])

    system["<b>Can't Run Alone</b><br/>[Software system]<br/>Matches runners by pace, area and schedule;<br/>schedules group runs at a pinned meeting point;<br/>checks weather and air quality"]

    osm["<b>OpenStreetMap tiles</b><br/>[External system]<br/>Map imagery"]
    weather["<b>Open-Meteo / CAMS</b><br/>[External system]<br/>Forecast and air quality"]
    messaging["<b>Email / SMS provider</b><br/>[External system]<br/>OTP and notifications<br/>(console adapter locally)"]

    visitor -->|"Registers, verifies OTP"| system
    runner -->|"Uses the web app (HTTPS, JSON)"| system
    moderator -->|"Moderates reports"| system
    runner -->|"Loads map tiles (browser)"| osm
    system -->|"Requests forecast and air quality (HTTPS)"| weather
    system -->|"Sends OTP and messages"| messaging

    classDef person fill:#08427b,stroke:#052e56,color:#ffffff
    classDef sys fill:#1168bd,stroke:#0b4884,color:#ffffff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#ffffff
    class visitor,runner,moderator person
    class system sys
    class osm,weather,messaging ext
```

## Notes

- A runner can act as an **organizer** (creates a run) or a **participant** (joins one); both are the same account.
- Map tiles are loaded by the browser directly from OpenStreetMap, not through the system ([0011](../adr/0011-render-maps-with-leaflet-and-openstreetmap-tiles.md)).
- The email/SMS provider is replaced by a clearly labeled console adapter in the local deployment; no real SMS is sent.
