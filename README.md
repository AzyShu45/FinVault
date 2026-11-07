# FinVault

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Java](https://img.shields.io/badge/Java-17%2B-blue.svg)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://www.docker.com/)

FinVault is a secure financial web application built to safely manage user accounts, financial records, and day-to-day transactions. To ensure that sensitive data remains strictly protected, it uses JWT authentication and role-based access control, backed by custom rate-limiting filters to prevent API abuse. Under the hood, the system is optimized with Caffeine caching for lightning-fast data retrieval, keeping performance smooth and responsive. Everything is tied together with a clean frontend using HTML, CSS, and JavaScript, making FinVault a reliable and user-friendly platform for handling everyday financial operations.

---

## ✨ Key Features

* **Authentication & Security:** Stateless authentication using JSON Web Tokens (JWT), password encryption, and secure password reset flows (`ForgotPassword` & `ResetPassword`).
* **Role-Based Access Control (RBAC):** Distinct permission boundaries based on user roles (`Role` enums) to protect administrative and user-specific endpoints.
* **API Rate Limiting:** Custom `RateLimitFilter` to protect endpoints from brute-force attacks and excessive API load.
* **High-Performance Caching:** Integrated Caffeine caching to speed up frequent queries and dashboard summaries.
* **Financial Record Management:** Full CRUD operations for tracking income and expenses (`RecordType`), complete with dashboard summaries.
* **Interactive UI:** Responsive client-side interface (`index.html`, `dashboard.html`, `records.html`, `users.html`) powered by modular JavaScript (`api.js`, `common.js`).
* **API Documentation:** Built-in OpenAPI / Swagger configuration (`OpenApiConfig`) for easy endpoint testing and exploration.
* **Cloud & Container Ready:** Includes a `Dockerfile` and `render.yaml` for seamless deployment.

---

## 🛠️ Tech Stack

* **Backend:** Java, Spring Boot, Spring Security, Spring Data JPA
* **Authentication:** JWT (JSON Web Tokens)
* **Caching & Performance:** Caffeine Cache, Custom Rate Limiting
* **Frontend:** HTML5, CSS3, Vanilla JavaScript
* **Build Tool:** Maven (`mvnw`)
* **DevOps & Quality:** Docker, GitHub Actions CI, Qodana Code Quality

---

## 🚀 Getting Started

### Prerequisites

* **Java 17+** installed on your machine
* **Maven** (or use the included Maven Wrapper `./mvnw`)
* **Docker** (optional, for containerized deployment)

### Running Locally

1. **Clone the repository:**
   ```bash
   git clone https://github.com/AzyShu45/FinVault.git
   cd FinVault
   ```

2. **Build and run the application using the Maven Wrapper:**
   * On Linux/macOS:
     ```bash
     ./mvnw spring-boot:run
     ```
   * On Windows:
     ```cmd
     mvnw.cmd spring-boot:run
     ```

3. **Access the application:**
   Open your browser and navigate to `http://localhost:8080`.

### Running with Docker

1. **Build the Docker image:**
   ```bash
   docker build -t finvault:latest .
   ```

2. **Run the container:**
   ```bash
   docker run -p 8080:8080 finvault:latest
   ```

---

## 📂 Project Structure

```text
FinVault/
├── .github/workflows/       # CI and Qodana code quality pipelines
├── src/
│   ├── main/
│   │   ├── java/com/finance/app/
│   │   │   ├── config/      # Security, JWT, Rate Limit, OpenAPI, and Seeder configs
│   │   │   ├── controller/  # REST Controllers (Auth, Dashboard, Records, Users)
│   │   │   ├── dto/         # Request and Response Data Transfer Objects
│   │   │   ├── entity/      # JPA Entities (User, FinancialRecord, PasswordResetToken)
│   │   │   ├── enums/       # Role and RecordType definitions
│   │   │   ├── exception/   # Global exception handling
│   │   │   ├── repository/  # Spring Data JPA Repositories
│   │   │   ├── security/    # JWT token generation and validation services
│   │   │   └── service/     # Core business logic
│   │   └── resources/
│   │       ├── static/      # Frontend HTML, CSS, and JS files
│   │       └── application.yml
│   └── test/                # Unit and Access Control integration tests
├── Dockerfile               # Docker container configuration
├── pom.xml                  # Maven dependencies and build configuration
├── render.yaml              # Render deployment blueprint
└── LICENSE                  # MIT License
```

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

Copyright (c) 2026 Azad Shukla
