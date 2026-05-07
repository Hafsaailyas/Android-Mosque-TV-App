flowchart TB
    %% Styling
    classDef android fill:#3DDC84,stroke:#000,color:#000
    classDef ui fill:#6200EE,stroke:#000,color:#fff
    classDef core fill:#FF9800,stroke:#000,color:#000
    classDef data fill:#2196F3,stroke:#000,color:#fff
    classDef storage fill:#4CAF50,stroke:#000,color:#fff
    classDef api fill:#9E9E9E,stroke:#000,color:#fff

    %% Startup Layer
    subgraph STARTUP["🚀 Startup"]
        A[📱 BootReceiver<br/>Listens: BOOT_COMPLETED]
        B[🚀 AppStartService<br/>Foreground + Max 3 retries]
    end

    %% UI Layer
    subgraph UI["📱 UI Layer"]
        C[🔐 LoginActivity<br/>• OTP Request<br/>• Login API Call<br/>• Spinner: Masjid/Device]
        D[📺 MainActivity<br/>• Lifecycle mgmt<br/>• Sync Timer Loop<br/>• Boot-to-Display]
    end

    %% Core Layer
    subgraph CORE["⚙️ Core Layer"]
        E[⚙️ GenFunc<br/>• syncRequest orchestration<br/>• Date/Time utilities<br/>• JSON parsing<br/>• QR generation]
        F[🎮 MainUI Singleton<br/>• DisplayObject list<br/>• Current prayer detection<br/>• Prayer timer (1s interval)]
    end

    %% Domain Layer
    subgraph DOMAIN["📦 Domain Layer"]
        K1[📋 Settings<br/>Masjid · Flags · Qiyamah]
        K2[📅 PerpetualCalendar<br/>30-day prayer times]
        K3[🖼️ Slides<br/>BLOB · Sequence · Duration]
        K4[🌐 Translations<br/>RTL/LTR · Urdu/English]
        K5[🎯 DisplayObject<br/>UI binding · Prayer window]
    end

    %% Data Layer
    subgraph DATA["💾 Data Layer"]
        G[🌐 VolleySingleton<br/>CustomJsonObjectRequest<br/>Bearer/Basic/No Auth]
        H[💾 DbFunc<br/>Insert · Update · Delete · Exists]
        M[🔑 AuthPreferences<br/>SharedPreferences wrapper]
    end

    %% Storage
    subgraph STORAGE["🗄️ Storage"]
        J1[(🗄️ SQLite: settings<br/>Single row config)]
        J2[(🗄️ SQLite: namaz_times<br/>PK: date · 30 days)]
        J3[(🗄️ SQLite: slides<br/>Full replace on sync)]
    end

    %% External
    I[☁️ MasjidWala API<br/>/sync · /screen_settings<br/>/calendar · /slideshow<br/>/translation · /login · /otp]

    %% Connections
    A --> B
    B -->|Launch| C
    C -->|Success| D
    C --> G
    C --> M

    D -->|Periodic| E
    D -->|Init| F

    E --> G
    E --> H
    E --> K1 & K2 & K3 & K4

    F --> H
    F --> E
    F --> K5

    G <--> I
    H --> J1 & J2 & J3
    M -.->|Provides token| G

    %% Styles
    class A,B android
    class C,D ui
    class E,F core
    class K1,K2,K3,K4,K5 data
    class G,H,M storage
    class I api
    class J1,J2,J3 storage
