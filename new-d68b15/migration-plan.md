# MIGRATION FROM SPRING BOOT APPLICATION CONFIGURATION TO ANSIBLE

The repository is not a Chef, Puppet, Salt, or PowerShell infrastructure-as-code repository. It is the Spring PetClinic sample application: a Java 17 Spring Boot 3.1 application built with Maven or Gradle, with local Docker Compose services for MySQL and PostgreSQL. Consequently, there are no source modules, cookbooks, manifests, recipes, or states to translate directly. The migration should create Ansible automation around application build/deployment, database provisioning, configuration, and operational validation rather than attempt a line-by-line conversion of application code.

Scope is small in module count but includes several operational concerns: a Spring Boot web application, selectable H2/MySQL/PostgreSQL database modes, database initialization scripts, static/template resources, and local container-based database environments. A proof-of-concept can be completed in approximately 1–2 weeks; a production-ready implementation with CI/CD integration, secrets management, TLS, backups, monitoring, and environment validation is more realistically 3–5 weeks, depending on the target platform.

## Module Migration Plan

This repository contains Spring Boot application components and Docker Compose configuration that need individual Ansible automation planning. It contains no identifiable Chef, Puppet, Salt, or PowerShell modules.

### MODULE INVENTORY

**CRITICAL PATH VERIFICATION:**
Only repository paths visible in the supplied tree and reviewed files are listed below.

- **petclinic-application**:
    - Description: Java 17 Spring Boot 3.1 web application providing the PetClinic user interface and HTTP endpoints for owners, pets, visits, veterinarians, validation, caching, and actuator operations.
    - Path: `src/main/java/org/springframework/samples/petclinic` and `src/main/resources`
    - Technology: Java / Spring Boot, deployed through Maven or Gradle
    - Key Features: Spring MVC and Thymeleaf, Spring Data JPA, Bean Validation, internationalized messages, static resources, cache support using Caffeine/JCache, and Spring Boot Actuator
    - Ansible target: Build or retrieve the JAR, install Java, create a dedicated service account, render external configuration, install a systemd unit, manage firewall/reverse-proxy integration, and perform health checks.

- **database-runtime**:
    - Description: Database runtime configuration supporting an in-memory H2 default and persistent MySQL or PostgreSQL profiles.
    - Path: `src/main/resources/application.properties`, `src/main/resources/application-mysql.properties`, `src/main/resources/application-postgres.properties`
    - Technology: H2, MySQL 8.0, and PostgreSQL 15.x; Spring datasource profiles
    - Key Features: Profile-specific JDBC URLs, username/password environment-variable overrides, idempotent SQL initialization, and disabled Hibernate schema generation (`ddl-auto=none`)
    - Ansible target: Provision the selected database, create the application database/user, deploy or execute schema/data initialization in a controlled phase, and render profile-specific datasource configuration.

- **database-schema-and-seed**:
    - Description: Vendor-specific schema and seed data used to initialize H2, MySQL, and PostgreSQL databases.
    - Path: `src/main/resources/db/h2`, `src/main/resources/db/mysql`, `src/main/resources/db/postgres`
    - Technology: SQL managed by Spring Boot startup initialization
    - Key Features: Separate schema/data files and vendor-specific setup documentation; the MySQL and PostgreSQL profiles enable initialization on startup.
    - Ansible target: Prefer an explicit database migration task or migration tool and run it once, rather than allowing every application start to mutate production data.

- **local-database-compose**:
    - Description: Local development orchestration for MySQL and PostgreSQL containers.
    - Path: `docker-compose.yml`
    - Technology: Docker Compose, MySQL 8.0, PostgreSQL 15.3
    - Key Features: Host port mappings on 3306 and 5432, bind-mounted MySQL configuration directory, and environment-based database credentials.
    - Ansible target: Treat as a development reference or replace with Ansible-managed Docker/container resources; do not assume it is a production topology.

### Infrastructure Files

- `pom.xml`: Primary Maven build, dependency and plugin manifest. It specifies Spring Boot 3.1.0, Java 17, database drivers, web dependencies, tests, code quality checks, CSS compilation, and Spring Boot image/build metadata. Ansible should normally deploy a tested artifact produced by CI rather than compile on every target host.
- `build.gradle`: Alternate Gradle build using Spring Boot 3.1.0 and Java 17. Establish which build system is authoritative before automating deployment.
- `mvnw`, `mvnw.cmd`, `gradlew`, `gradlew.bat`, `gradle/wrapper`: Build wrappers for developer/CI reproducibility; they are not server configuration modules.
- `docker-compose.yml`: Local MySQL/PostgreSQL development services. Its credentials and permissive settings require replacement for shared or production environments.
- `src/main/resources/application.properties`: Default H2 configuration, SQL locations, Thymeleaf, JPA, logging, actuator exposure, and static-resource caching settings.
- `src/main/resources/application-mysql.properties`: MySQL profile and environment-variable datasource overrides.
- `src/main/resources/application-postgres.properties`: PostgreSQL profile and environment-variable datasource overrides.
- `src/main/resources/db/*`: Database schemas, seed data, and setup notes that must be included in the deployment/data migration design.
- `src/main/java/org/springframework/samples/petclinic/PetClinicApplication.java`: Spring Boot application entry point. The deployment service should invoke the built JAR rather than replace this code with Ansible tasks.
- `readme.md`: Documents Java 17+, local execution, Maven/Gradle builds, profiles, Docker database examples, and CSS generation. Use it as a baseline for acceptance tests and operational documentation.

### Target Details

- **Operating System**: Linux is the most likely target because the repository documents `./mvnw`, `./gradlew`, Docker usage, and Unix-style paths. The exact distribution is not specified; standardize on Red Hat Enterprise Linux 9 for the initial Ansible design unless the hosting platform dictates otherwise. The playbook should also document package-manager differences if Ubuntu support is required.
- **Virtual Machine Technology**: Not specified. No VMware, VirtualBox, KVM, Hyper-V, cloud-init, or guest-tools configuration is present.
- **Cloud Platform**: Not specified. No AWS, Azure, or GCP-specific configuration is present. Keep cloud integrations in a separate inventory/group-vars layer.

## Migration Approach

1. **Define the target topology and ownership**: Decide whether Ansible manages one host containing the application and database, separate application/database hosts, or containers. Decide whether databases are self-managed or supplied as managed services.
2. **Standardize artifact production**: Select Maven or Gradle as the CI build path, run the existing tests and checks, publish a versioned JAR/container artifact, and deploy that immutable artifact with Ansible.
3. **Create Ansible roles**: Suggested roles are `java_runtime`, `petclinic_app`, `petclinic_database`, `petclinic_schema`, `reverse_proxy` (if required), and `observability`. Keep environment values in inventory/group variables and secrets in Ansible Vault or an external secret manager.
4. **Externalize runtime configuration**: Render a restricted `application-<profile>.properties` or environment file. Set the active profile, datasource URL, credentials, logging, actuator exposure, and allowed bind addresses per environment.
5. **Provision and initialize the database**: Install the chosen engine or connect to a managed service, create least-privilege users, apply schema/data in an explicit ordered step, and establish backups before production cutover.
6. **Deploy and operate the service**: Install Java 17, create a non-root service account, copy the artifact, create a systemd service, configure restart and resource limits, open only required ports, and validate the application endpoint and database connectivity.
7. **Validate parity and hand over**: Run unit/integration tests, profile-specific startup tests for H2/MySQL/PostgreSQL as applicable, smoke tests for owner/pet/vet flows, rollback tests, and operational runbooks. Coordinate application, database, security, and platform owners through review checkpoints.

### Key Dependencies to Address

- **Java 17**: Install and pin a supported JDK/JRE package with Ansible; verify the runtime used by systemd matches CI.
- **Spring Boot 3.1.0**: Preserve the tested application/runtime combination initially, then plan framework upgrades separately.
- **Maven or Gradle**: Choose one authoritative build path. Prefer CI builds and artifact promotion over compiling on production hosts.
- **H2 runtime**: Retain for development/test only unless there is an explicit requirement for an in-memory deployment.
- **MySQL 8.0 / PostgreSQL 15.3**: Provision through the appropriate Ansible collection or use managed database services. Pin versions and define upgrade/backup procedures.
- **JDBC drivers**: Already packaged as runtime dependencies; ensure the deployed artifact contains the required driver for the selected profile.
- **Bootstrap 5.2.3, Font Awesome 4.7.0, Caffeine, JCache, Thymeleaf, Actuator**: Application dependencies delivered by the build; Ansible should not independently install them.
- **Docker Compose**: Use only for local development unless container deployment is explicitly selected. If retained, manage it with a dedicated container role and secure environment files.
- **CSS build tooling**: Maven’s `css` profile uses WebJars and libsass; run it in CI before packaging. The README notes no equivalent Gradle CSS profile.

### Security Considerations

- **Database credentials**: `docker-compose.yml` exposes `petclinic` credentials and permits an empty MySQL root password. The profile files also show default credentials (`petclinic`) and environment variables `MYSQL_PASS`/`POSTGRES_PASS`. Remove defaults from shared environments, rotate any reused credentials, and store values in Ansible Vault or a secret manager.
- **MySQL exposure**: Compose publishes port 3306 and enables `MYSQL_ALLOW_EMPTY_PASSWORD=true`. Remove this setting, bind databases to private interfaces, restrict firewall access to the application tier, and use TLS where required.
- **PostgreSQL exposure**: Compose publishes port 5432. Restrict network access and configure authentication, TLS, and host-based access rules for the target environment.
- **Actuator exposure**: `management.endpoints.web.exposure.include=*` exposes all actuator endpoints by default. Limit endpoints, isolate management traffic, require authentication, and avoid exposing environment/configuration or shutdown-style operations publicly.
- **Application execution**: Run the JAR as a dedicated non-root user with a hardened systemd unit, restrictive file permissions, controlled Java options, and no secrets in command-line arguments or world-readable files.
- **Transport security**: The repository documents HTTP on port 8080 and contains no TLS/reverse-proxy configuration. Add TLS termination, certificate lifecycle management, secure headers, and HTTP-to-HTTPS policy at the chosen ingress layer.
- **SQL initialization**: Startup initialization is enabled for MySQL and PostgreSQL. In production, prevent accidental reseeding or schema changes on every restart; use an audited migration process and backup/rollback plan.
- **Secrets inventory limitation**: No Chef encrypted data bags, Puppet Hiera secrets, certificates, or cloud secret references are present. The visible plaintext/default database credentials are nevertheless a security finding and must be treated as credentials until proven otherwise.

### Technical Challenges

- **No existing infrastructure model**: There is no source automation to translate, so host groups, network boundaries, service ownership, patching, backups, and deployment strategy must be designed from scratch.
- **Dual build systems**: Maven and Gradle describe largely overlapping builds, while CSS generation is documented only for Maven. Choose one canonical pipeline and test artifact equivalence before migration.
- **Database engine divergence**: H2, MySQL, and PostgreSQL have separate SQL trees and operational behavior. Validate schema, seed data, JDBC settings, collation, timezone, and transaction behavior per supported engine.
- **Development versus production ambiguity**: Docker Compose is clearly local-oriented, but the target runtime is unspecified. Confirm whether Ansible manages VMs, bare metal, containers, or only external services.
- **Application startup data behavior**: Spring SQL initialization and `ddl-auto=none` require ordered database readiness and initialization. Add idempotency and migration locking rather than relying on concurrent application startup.
- **Operational gaps**: No deployment unit, reverse proxy, TLS, backup, monitoring alert policy, log rotation, resource limits, or rollback procedure is included. These must be supplied by the Ansible implementation and platform team.
- **Actuator and health validation**: The application has Actuator but no target-specific security policy. Define safe liveness/readiness endpoints and use them in rolling deployment checks.

### Migration Order

1. **Foundation and decisions**: Confirm target OS/topology, authoritative build (Maven or Gradle), supported database (one primary target first), ownership, networking, and secret-management standards.
2. **Build and artifact pipeline**: Build Java 17 artifacts, run existing tests/checkstyle/coverage, package CSS, and publish a versioned artifact.
3. **`java_runtime` and `petclinic_app` roles**: Install Java, service account, directories, systemd unit, externalized configuration, logging, and basic health checks. Validate with H2 first for low-risk application deployment.
4. **`petclinic_database` role**: Provision MySQL or PostgreSQL with hardened authentication, private networking, least-privilege application access, and backups. Select one persistent database as the first production path.
5. **`petclinic_schema` role**: Apply the selected vendor schema and data explicitly, then test restart/redeploy idempotency and rollback procedures.
6. **Security and ingress hardening**: Add TLS, firewall rules, restricted actuator access, secret rotation, systemd hardening, and reverse proxy/load-balancer integration.
7. **Secondary database profile and local Compose alignment**: Validate the remaining supported database profile and update developer documentation/Compose defaults without weakening production controls.
8. **Production cutover and handover**: Perform staged deployment, smoke/load testing using the existing JMeter plan, backup verification, rollback rehearsal, and transfer runbooks to operations.

### Assumptions

- The requested migration means creating Ansible-based deployment and environment automation around this application, not converting Java source code into Ansible.
- The supplied tree is complete for the relevant source and infrastructure inventory; no hidden CI, inventory, deployment, Terraform, Kubernetes, or secret files were provided.
- There are zero Chef cookbooks, Puppet modules, Salt states, or PowerShell modules in the supplied repository, so no individual source-IaC module mapping is possible.
- The target operating system, VM/container platform, cloud provider, network topology, DNS, ingress, and database hosting model are unspecified.
- RHEL 9 is used as the planning default only because the required format calls for a default when the source is unclear; it must be confirmed before implementation.
- Java 17 remains the target runtime and Spring Boot 3.1.0 remains unchanged for the first migration wave.
- The team will choose either Maven or Gradle as the canonical build path; the plan does not assume both must run in production.
- Database credentials visible in Compose and properties are development examples, but they must be rotated and treated as compromised if reused.
- The application’s default H2 mode is suitable for initial smoke tests, not necessarily for production.
- SQL initialization files are intended to be applied in a controlled migration step for production, even though the application currently supports startup initialization.
- The existing JMeter test plan is available for performance validation, but its acceptance thresholds and environment assumptions were not reviewed.
- No high-availability, disaster-recovery, data-retention, compliance, SLO, or capacity requirements are defined; those are prerequisites for production sizing and a final timeline.
- A realistic schedule is 1–2 weeks for a basic single-environment deployment and 3–5 weeks for hardened multi-environment automation, subject to the unresolved platform and security decisions.
