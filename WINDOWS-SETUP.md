## Windows-Side Setup for Power BI Desktop Bridge

This project uses a dual-setup strategy:
- **Linux**: OpenCode CLI + MCP server for offline PBIP/PBIR editing (modeling, DAX, linting, audits)
- **Windows**: Power BI Desktop June 2026+ with Desktop Bridge for live visualization and hot-reload

### Prerequisites

- Windows 10/11
- **Power BI Desktop June 2026 or later**
- **Python 3.10+** (any Python, doesn't need to match Linux version)
- **Git**
- **OpenCode CLI** (if you want to run the same workflow on Windows)

### Step 1: Sync the project to Windows

Clone or copy this entire project directory (`powerbi-contract-uaepl/`) to your Windows machine.

### Step 2: Enable Desktop Bridge in Power BI Desktop

1. Open Power BI Desktop
2. Go to **File > Options and settings > Options**
3. Under **Preview features**, enable:
   - **"Enable external tool access to Power BI Desktop through secure local APIs"** (on by default in June 2026+)
   - **"Store reports using enhanced metadata format (PBIR)"** — enables the PBIR report format for programmatic authoring
4. Click **OK** and restart Power BI Desktop

### Step 3: Install ADOMD.NET for live connectivity

```powershell
# Option A: Install SQL Server Management Studio (includes ADOMD.NET)
# Download: https://aka.ms/ssmsfullsetup

# Option B: Download ADOMD.NET + TOM from NuGet manually
mkdir C:\pbi-dlls
Invoke-WebRequest -Uri "https://www.nuget.org/api/v2/package/Microsoft.AnalysisServices.AdomdClient.retail.amd64" -OutFile "$env:TEMP\adomd.nupkg"
Invoke-WebRequest -Uri "https://www.nuget.org/api/v2/package/Microsoft.AnalysisServices.retail.amd64" -OutFile "$env:TEMP\amo.nupkg"
Expand-Archive "$env:TEMP\adomd.nupkg" "$env:TEMP\adomd"
Expand-Archive "$env:TEMP\amo.nupkg" "$env:TEMP\amo"
Copy-Item "$env:TEMP\adomd\lib\net45\*.dll" C:\pbi-dlls\
Copy-Item "$env:TEMP\amo\lib\net45\*.dll" C:\pbi-dlls\
```

### Step 4: Install full Python dependencies

```powershell
cd C:\path\to\powerbi-contract-uaepl\powerbi-mcp
pip install -r requirements.txt
```

### Step 5: Set environment variables (optional)

```powershell
$env:ADOMD_DLL_PATH = "C:\pbi-dlls"
$env:TOM_DLL_PATH = "C:\pbi-dlls"
```

Or add them to the `environment` block in `opencode.json` on the Windows side.

### Step 6: Create your first PBIP project

Open Power BI Desktop and save as PBIP:

1. Create a new report (or open an existing one)
2. **File > Save as**
3. Choose **"Power BI Project (*.pbip)"** as the file type
4. Save into your project directory

This creates the `.pbip` folder structure with `model/` and `report/` subdirectories.

### The Edit-and-Verify Workflow

On either machine (Linux or Windows), you can edit the PBIP files. Then on Windows:

```
# In OpenCode (or any MCP client):
"You are connected to the powerbi MCP server."

"Load the PBIP project at C:/path/to/MyReport.Report"
"Add an 'Overview' page with a bar chart of Contract Value by Type"

# Check Desktop Bridge status
"bridge_status"

# Hot-reload the running Desktop (no restart needed)
"bridge_reload"

# Take a PNG screenshot to verify visually
"bridge_screenshot"
```

### opencode.json (Windows variant)

The Windows opencode.json should include paths to the ADOMD.NET DLLs:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "powerbi": {
      "type": "local",
      "command": ["python", "src/server.py"],
      "cwd": "./powerbi-mcp",
      "environment": {
        "PYTHONPATH": "./powerbi-mcp/src",
        "POWERBI_MCP_READONLY": "false",
        "ADOMD_DLL_PATH": "C:\\pbi-dlls",
        "TOM_DLL_PATH": "C:\\pbi-dlls"
      },
      "timeout": 30000
    }
  }
}
```

### Limitations

- **`bridge_screenshot`** may return internal errors on current Desktop builds (a known Desktop-side preview bug). Status, manifest, and hot-reload work reliably.
- **Live TOM writes** require the TOM assembly (from NuGet above).
- **Live XMLA/cloud paths** need a premium capacity or Fabric workspace.
