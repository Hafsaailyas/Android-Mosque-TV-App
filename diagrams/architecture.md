flowchart TD
    %% ── Styling Definition ──────────────────────────────────────────
    classDef android fill:#3DDC84,stroke:#000,color:#000
    classDef ui fill:#6200EE,stroke:#000,color:#fff
    classDef core fill:#FF9800,stroke:#000,color:#000
    classDef domain fill:#2196F3,stroke:#000,color:#fff
    classDef data fill:#4CAF50,stroke:#000,color:#fff
    classDef api fill:#9E9E9E,stroke:#000,color:#fff
    classDef storage fill:#EF4444,stroke:#000,color:#fff

    %% ── Startup Layer ────────────────────────────────────────────────
    subgraph STARTUP["🚀 Startup Layer"]
        A["📱 BootReceiver<br/>Listens: BOOT_COMPLETED"]
        B["🚀 AppStartService<br/>Foreground + Max 3 retries"]
    end

    %% ── UI / Presentation Layer ──────────────────────────────────────
    subgraph UI["📱 UI / Presentation Layer"]
        C["🔐 LoginActivity<br/>• OTP Request<br/>• Login API Call<br/>• Spinner: Masjid/Device"]
        D["📺 MainActivity<br/>• Lifecycle management<br/>• Sync Timer Loop<br/>• Boot-to-Display"]
        F["🎮 MainUI Singleton<br/>• DisplayObject list<br/>• Current prayer detection<br/>• Prayer timer (1s interval)"]
    end

    %% ── Core / Orchestration Layer ───────────────────────────────────
    subgraph CORE["⚙️ Core / Orchestration Layer"]
        E["⚙️ GenFunc<br/>• syncRequest orchestration<br/>• Date/Time utilities<br/>• JSON parsing<br/>• QR generation"]
    end

    %% ── Domain Layer ─────────────────────────────────────────────────
    subgraph DOMAIN["📦 Domain Layer"]
        K1["📋 Settings<br/>Masjid · Flags · Qiyamah"]
        K2["📅 PerpetualCalendar<br/>30-day prayer times"]
        K3["🖼️ Slides<br/>BLOB · Sequence · Duration"]
        K4["🌐 Translations<br/>RTL/LTR · Urdu/English"]
        K5["🎯 DisplayObject<br/>UI binding · Prayer window"]
    end

    %% ── Data Layer ───────────────────────────────────────────────────
    subgraph DATA["💾 Data Layer"]
        G["🌐 VolleySingleton<br/>CustomJsonObjectRequest<br/>Bearer/Basic/No Auth"]
        H["💾 DbFunc<br/>Insert · Update · Delete · Exists"]
        M["🔑 AuthPreferences<br/>SharedPreferences wrapper"]
    end

    %% ── Storage Layer ────────────────────────────────────────────────
    subgraph STORAGE["🗄️ Storage Layer (SQLite)"]
        J1["settings<br/>Single row config"]
        J2["namaz_times<br/>PK: date · 30 days"]
        J3["slides<br/>Full replace on sync"]
    end

    %% ── Remote API ───────────────────────────────────────────────────
    I["☁️ MasjidWala API<br/>/sync · /screen_settings<br/>/calendar · /slideshow<br/>/translation · /login · /otp"]

    %% ── Connections / Flow ───────────────────────────────────────────
    A -->|"BOOT_COMPLETED"| B
    B -->|"Launch after 5s"| C
    C -->|"on auth success"| D
    C -->|"OTP / Login"| G
    C -->|"savePreference"| M

    D -->|"Periodic syncRequest()"| E
    D -->|"init(context)"| F

    E -->|"addToRequestQueue (BEARER_TOKEN)"| G
    E -->|"insert / upsert"| H
    E -->|"constructs"| K1 & K2 & K3 & K4
    E -->|"saveTranslation"| M

    F -->|"reads settings & prayer times"| H
    F -->|"time util calls"| E
    F -->|"builds list"| K5

    G <-->|"HTTP (Bearer/Basic/None)"| I

    H -->|"SQL write"| J1 & J2 & J3
    H -->|"SQL read"| K1 & K2 & K3

    M -.->|"provides token"| G
    M -->|"returns serialised JSON"| K4

    %% ── Apply Styles ─────────────────────────────────────────────────
    class A,B android
    class C,D,F ui
    class E core
    class K1,K2,K3,K4,K5 domain
    class G,H,M data
    class J1,J2,J3 storage
    class I api
