# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Danphe EMR — an open-source hospital management system (HIMS + EMR + EHR) used in production by 60+ hospitals. Two-tier app: an ASP.NET Core backend (C#, running on .NET Framework 4.6.1) serving a REST API, and an Angular 7 SPA frontend. Data lives in SQL Server.

Developer setup guide: https://opensource-emr.github.io/hospital-management-emr/#setup

## Build & Run

### Backend (Windows only — targets net461, build with Visual Studio or MSBuild)

- Solution: `Code/Solutions/DanpheEMR.sln`
- Web project: `Code/Websites/DanpheEMR/DanpheEMR.csproj` (ASP.NET Core 2.0 on net461, Kestrel + IIS integration)
- Run via Visual Studio (IIS Express profile, http://localhost:56326/Account/Login) or `msbuild Code/Solutions/DanpheEMR.sln`
- NuGet packages are restored to `Code/Solutions/packages/` (packages.config style for class libraries); reference DLLs in `Code/ReferenceDLLs/`

### Frontend (Angular 7, in `Code/Websites/DanpheEMR/wwwroot/DanpheApp`)

```bash
cd Code/Websites/DanpheEMR/wwwroot/DanpheApp
npm install
npm run build       # ng build with --max-old-space-size=16384 (build is memory-hungry)
npm start           # ng serve (dev server on :4200)
npm test            # Karma/Jasmine unit tests
npm run lint        # tslint via ng lint
ng test --include='**/some.component.spec.ts'   # single test file
```

The Angular build outputs to `dist/DanpheApp` with `deployUrl: /DanpheApp/dist/DanpheApp/` — the compiled bundle is served as static files by the ASP.NET app (the SPA is hosted inside `Views/Home/AppMain.cshtml` after MVC login at `Views/Account/Login.cshtml`). After changing Angular code you must rebuild for the served app to update; `ng serve` alone doesn't go through the .NET host/auth.

### Database (SQL Server)

Three databases, connection strings in `Code/Websites/DanpheEMR/appsettings.json` (`Connectionstring` = main EMR db, `ConnectionStringAdmin` = DanpheAdmin, `ConnectionStringPACSServer` = PACS). Creation scripts live in `Database/`:
- `Database/1. Admin-Db/1. DanpheAdmin_CompleteDB.sql`
- `Database/2. EMR-Db/DanpheInternationalDB/` (run scripts in numbered order)
- `Database/CleanUpScript.sql` to reset

## Architecture

### Solution layout (`Code/`)

- `Websites/DanpheEMR/` — the web app: ASP.NET Core MVC controllers (API) + the Angular SPA under `wwwroot/DanpheApp`
- `Components/` — .NET Framework class libraries:
  - `DanpheEMR.ServerModel` — entity/model classes (the largest project)
  - `DanpheEMR.DalLayer` — Entity Framework 6 (not EF Core) data access; one DbContext per module (`BillingDbContext`, `LabDbContext`, `PharmacyDbContext`, …) plus raw SQL helpers in `DALFunctions.cs`
  - `DanpheEMR.Core`, `DanpheEMR.Security` (auth/session), `DanpheEMR.Sync`, `DanpheEMR.Jobs`, `DanpheEMR.AccTransfer`
- `Utilities/` — side tools: `ServerSidePrinter` (separate solution), `TestingPlayGroundConsole`

### Module-per-folder convention

The system is ~20+ hospital modules (Billing, Pharmacy, Lab, Radiology, ADT/Admission, Emergency, Nursing, Inventory, Accounting, OT, Insurance, …). Each module appears in three parallel places:

1. `Code/Websites/DanpheEMR/Controllers/<Module>/` — API controllers plus `<Module>BL.cs` business-logic classes
2. `Code/Components/DanpheEMR.DalLayer/<Module>DbContext.cs` — EF6 context
3. `Code/Websites/DanpheEMR/wwwroot/DanpheApp/src/app/<module>/` — Angular feature module (most are lazy-loaded; routing constants in `app-routing.constant.ts`)

When adding a feature, follow this layering: Angular service → controller endpoint → BL class → DbContext/ServerModel entity.

### Backend specifics

- `Startup.cs` / `ConfigureServices.cs` / `DependencyInjection/` wire up MVC, JWT bearer auth (`JwtTokenConfig` in appsettings.json), SignalR (e.g. `/FonePayTransactionStatus` hub for POS payments), and rate limiting (`RateLimit/`)
- Serilog configured from `logging.Configuration.json` (not appsettings.json)
- `appsettings.json` carries many feature flags (`EnableFewaPay`, `RealTimeSSFClaimBooking`, `IsDemoEnvironment`, …) and machine-specific paths — don't commit local connection-string/path changes
- Audit logging via Audit.EntityFramework (`IsAuditEnable` flag)
- The csproj uses `EnableDefaultContentItems=false` and explicit `<None Include>` entries for controller files — new backend files may need explicit csproj entries

### Frontend specifics

- Angular 7 + TypeScript 3.1, RxJS 6 (with rxjs-compat); ag-Grid for data tables, jQuery/Bootstrap-era plugins under `wwwroot/assets/` (don't touch those vendored libs)
- Shared/cross-cutting code: `src/app/shared/`, `src/app/core/`, `src/app/common/`; security/session in `src/app/security/`
- LIS (lab machine) integration and DICOM viewing (cornerstone) are part of the frontend
