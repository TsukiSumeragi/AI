# FLOW SISTEM

```mermaid
graph TD
    %% --- Start/End Nodes ---
    Start(["MULAI: Pengguna Membuka Aplikasi (Streamlit)"])
    End(["SELESAI: Jadwal & Rute Ditampilkan di Dasbor"])

    %% --- Input Sections ---
    subgraph Input_Data ["Input Data"]
        InputDepot["Input Koordinat Depot Awal"]
        InputDest["Dataset CSV (id, latitude, longitude, demand)"]
        InputVehCap["Input Batas Kapasitas Muatan Armada"]
        InputHazard["Dataset GeoJSON Poligon Zona Macet (Tol Bitung)"]
    end

    %% --- Process Section - Core AI (DEAP) ---
    subgraph GA_Engine ["Proses AI (Algoritma Genetika)"]
        InitPop["Inisialisasi Kromosom (Giant TSP Tour)"]
        GenLimit{"Kriteria Berhenti<br/>Terpenuhi?"}
        
        subgraph GA_Loop ["Siklus Evolusi"]
            SplitAlg["Pemotongan Kromosom Berdasarkan Kapasitas Kendaraan"]
            EvalFitness["Evaluasi Fitness (Jarak Pendek & Hemat BBM)"]
            CheckHazard{"Rute Memotong Poligon Tol Bitung?"}
            AddPenaltyH["Turunkan Nilai Fitness (Penalti Berat)"]
            CrossoverOp["Crossover (Kawin Silang Antar Rute Terbaik)"]
            MutationOp["Mutasi (Modifikasi Acak Urutan Titik Kunjungan)"]
            CreateOffspring["Hasilkan Generasi Baru"]
        end
    end

    %% --- Output/Visualization Section ---
    subgraph Output_Vis ["Output (Streamlit & Folium)"]
        GetBestR["Ekstrak Jadwal Rute Kurir Terbaik & Estimasi Waktu"]
        RenderMap["Visualisasi Garis Lintasan pada Peta Digital"]
    end

    %% --- Connectors ---
    Start --> Input_Data
    Input_Data --> InitPop
    InitPop --> GenLimit
    
    %% Loop Logic
    GenLimit -- Tidak --> SplitAlg
    SplitAlg --> EvalFitness
    EvalFitness --> CheckHazard
    CheckHazard -- Ya --> AddPenaltyH
    AddPenaltyH --> CrossoverOp
    CheckHazard -- Tidak --> CrossoverOp
    CrossoverOp --> MutationOp
    MutationOp --> CreateOffspring
    CreateOffspring --> GenLimit
    
    %% Termination Logic
    GenLimit -- Ya --> GetBestR
    GetBestR --> RenderMap
    RenderMap --> End

    %% Styling
    classDef process fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef decision fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef inputout fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef loopsub fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px,stroke-dasharray: 5 5;
    classDef startend fill:#ffccbc,stroke:#bf360c,stroke-width:2px,rx:10,ry:10;

    class Start,End startend;
    class InputDepot,InputDest,InputVehCap,InputHazard,GetBestR,RenderMap inputout;
    class InitPop,SplitAlg,EvalFitness,AddPenaltyH,CrossoverOp,MutationOp,CreateOffspring process;
    class GenLimit,CheckHazard decision;
    class GA_Loop loopsub;
```

# UML

```mermaid
classDiagram
    %% --- Stereotype Definitions ---
    class `Boundary (Streamlit)` {
        <<Boundary>>
    }
    class `Controller (GA Engine)` {
        <<Control>>
    }
    class `Entity (Model)` {
        <<Entity>>
    }
    class `Utility` {
        <<Utility>>
    }

    %% --- Class Definitions ---

    class LogisticsDashboard {
        -streamlit.sidebarInput inputs
        -folium.Map mainMap
        -List~RouteStop~ bestRoutes
        +displayDashboard()
        +getInputs() void
        +renderMap(List~RouteStop~ routes) folium.Map "SKPL-F07"
        +showMetrics(dist, cost) void
        +handleOptimizeButtonClick() void
    }

    class AlgorithmController {
        -List~Node~ nodes
        -double vehicleCapacity
        -List~HazardZone~ hazards
        -GeneticAlgorithmParams gaParams
        +runOptimization() void "SKPL-F05"
        +initializePopulation() void
        +runSplitAlgorithm(List~Node~ giantTour) List~Route~
        +calculateFitness(Individual ind) double
        +calculateLoadFuelModel(Route r) double "LFCM"
    }

    class GeneticAlgorithm {
        -List~Individual~ population
        -double crossoverRate
        -double mutationRate
        +evaluatePopulation() void
        +selectTournament() Individual
        +performCrossover(Individual parent1, Individual parent2) Individual "Order Crossover"
        +performMutation(Individual ind) void "Swap Mutation"
        +checkTermination() boolean
    }

    class Individual {
        -List~int~ chromosome "Permutation Encoding (Giant TSP Tour)"
        -double fitnessScore
        -List~Route~ routesCVRP
        +getFitness() double
        +setFitness(double score) void
    }

    class Node {
        -int id
        -double latitude
        -double longitude
        -double demandKg "SKPL-F02"
        -boolean isDepot "SKPL-F01"
        +getCoordinates() Tuple~double~
    }

    class Vehicle {
        -int id
        -double capacityKg
        -double emptyWeightKg
        -double fuelModel
    }

    class HazardZone {
        -string name
        -List~Tuple~double~~ polygonCoords "GeoJSON Polygon"
        -double penaltyWeight "w_hz"
        +intersects(List~Tuple~double~~ polyline) boolean
    }

    class Route {
        -int vehicleId
        -List~Node~ orderedNodes
        -double totalDistance
        -double totalLoad
        +addNode(Node n) void
    }

    class CalculatorUtilities {
        +haversineDistance(Lat1, Lon1, Lat2, Lon2) double
        +geodesicDistance(Lat1, Lon1, Lat2, Lon2) double "SKPL-NF01"
        +loadDependentFuel(Route route, Vehicle vehicle) double "PRP Model"
    }

    %% --- Stereotype Assignments ---
    %% Assign classes to their respective stereotyps for visualization
    <<Boundary>> LogisticsDashboard
    <<Control>> AlgorithmController
    <<Control>> GeneticAlgorithm
    <<Entity>> Individual
    <<Entity>> Node
    <<Entity>> Vehicle
    <<Entity>> HazardZone
    <<Entity>> Route
    <<Utility>> CalculatorUtilities

    %% --- Relationships ---
    %% Association (Uses)
    LogisticsDashboard ..> AlgorithmController : "Triggers"
    LogisticsDashboard ..> CalculatorUtilities : "Uses for final display"
    AlgorithmController ..> GeneticAlgorithm : "Orchestrates"
    AlgorithmController ..> CalculatorUtilities : "Uses for matrices & fuel"

    %% Composition (Part of)
    GeneticAlgorithm "1" *-- "many" Individual : "Maintains Population"
    Individual "1" *-- "many" Route : "Consists of (Split Result)"
    Route "1" *-- "many" Node : "Services"
    AlgorithmController "1" *-- "many" Node : "Processes"
    AlgorithmController "1" *-- "many" HazardZone : "Considers for Penalty"
    AlgorithmController "1" *-- "many" Vehicle : "Constraints by"
```

# SKEMA BASIS DATA

```mermaid
erDiagram
    %% --- Entities ---

    %% Data Induk / Konfigurasi
    COMPANY {
        int company_id PK
        string name
        decimal depot_latitude
        decimal depot_longitude
    }

    VEHICLE {
        int vehicle_id PK
        int company_id FK "FK_Company_Vehicle"
        string vehicle_name
        decimal capacity_kg "SKPL-F03 - Batas Muatan"
        decimal empty_weight_kg "LFCM Model Param"
        decimal fuel_efficiency "Liters/Km"
    }

    NODE {
        int node_id PK
        int company_id FK "FK_Company_Node"
        string node_name
        decimal latitude
        decimal longitude
        decimal demand_kg "SKPL-F02 - Bobot Paket"
        boolean is_depot
    }

    HAZARD_ZONE {
        int zone_id PK
        int company_id FK "FK_Company_Hazard"
        string zone_name "e.g., Jagabita Area"
        decimal penalty_weight "w_hz - SKPL-F04"
    }

    %% Sub-table for polygon vertices (Folium GeoJSON map data)
    HAZARD_VERTEX {
        int vertex_id PK
        int zone_id FK "FK_Hazard_Vertex"
        decimal latitude
        decimal longitude
        int sequence_order
    }

    %% Data Transaksional / Hasil Optimasi
    ROUTE_OPTIMIZATION {
        int optimization_id PK
        int company_id FK "FK_Company_Opt"
        datetime execution_time
        int total_nodes_processed
        decimal total_fleet_distance_km
        decimal total_fleet_fuel_liters
        decimal final_fitness_score "GA Evaluation"
    }

    %% Individual routes within a single optimization (from Split Algorithm)
    OPTIMIZED_ROUTE {
        int route_id PK
        int optimization_id FK "FK_Opt_Route"
        int vehicle_id FK "FK_Vehicle_Route"
        int route_sequence "Sequential Order (1, 2, ...)"
        decimal total_distance_km
        decimal total_load_kg
        decimal estimated_fuel_consumed "LFCM Model"
    }

    %% Order of nodes within a specific optimized route (Polyline Folium)
    ROUTE_STOP {
        int stop_id PK
        int route_id FK "FK_Route_Stop"
        int node_id FK "FK_Node_Stop"
        int stop_order "Sequential Order (1, 2, ...)"
        decimal current_vehicle_load "KG (Reduces along route)"
    }

    %% --- Relationships ---
    COMPANY ||--|{ VEHICLE : "owns"
    COMPANY ||--|{ NODE : "manages"
    COMPANY ||--|{ HAZARD_ZONE : "defines"
    COMPANY ||--|{ ROUTE_OPTIMIZATION : "executes"

    HAZARD_ZONE ||--|{ HAZARD_VERTEX : "consists_of (Polygon)"

    ROUTE_OPTIMIZATION ||--|{ OPTIMIZED_ROUTE : "results_in (Split Alg)"
    
    VEHICLE ||--|{ OPTIMIZED_ROUTE : "assigned_to"
    
    OPTIMIZED_ROUTE ||--|{ ROUTE_STOP : "contains_sekuensial"
    NODE ||--|{ ROUTE_STOP : "serviced_by"
```
