flowchart TD
    subgraph OS["Android OS"]
        BOOT["BootReceiver"]
        SVC["AppStartService"]
    end

    subgraph PRES["Presentation"]
        LOGIN["LoginActivity"]
        MAIN["MainActivity"]
        MAINUI["MainUI"]
    end

    subgraph ORCH["Orchestration"]
        GEN["GenFunc"]
    end

    subgraph DOM["Domain"]
        S["Settings"]
        PC["PerpetualCalendar"]
        SL["Slides"]
        DO["DisplayObject"]
        PR["Prayer"]
        TR["Translations"]
    end

    subgraph DATA["Data"]
        DB["DbFunc"]
        VS["VolleySingleton"]
        AP["AuthPreferences"]
    end

    subgraph SQLITE["SQLite"]
        T1["settings"]
        T2["namaz_times"]
        T3["slides"]
    end

    subgraph API["Remote API"]
        EP["REST Endpoints"]
    end

    BOOT -->|"BOOT_COMPLETED"| SVC
    SVC --> LOGIN
    LOGIN --> MAIN
    LOGIN --> VS
    LOGIN --> AP

    MAIN --> GEN
    MAIN --> MAINUI
    MAINUI --> DB
    MAINUI --> DO

    GEN --> VS
    GEN --> DB
    GEN --> AP
    GEN --> S & PC & SL & TR

    VS <-->|"HTTP"| EP

    DB --> T1 & T2 & T3
    DB --> S & PC & SL
    AP --> TR
    DO --> PR
