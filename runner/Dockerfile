# escape=`
ARG LTSC_YEAR=2022
FROM mcr.microsoft.com/windows/servercore:ltsc${LTSC_YEAR} as runner
SHELL [ "powershell.exe", "-NoLogo", "-Command", "$ErrorActionPreference='Stop';", "$ProgressPreference='SilentlyContinue';" ]
# runner, auto download latest release
RUN Invoke-WebRequest https://nodejs.org/dist/v16.15.1/node-v16.15.1-win-x64.zip -OutFile node.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory(\"$PWD/node.zip\", \"$env:temp\node\"); `
    mv \"$env:temp\node\node-v16.15.1-win-x64\" c:\node\; `
    $env:Path=\"c:\node\;$env:PATH\"; `
    $releases = Invoke-RestMethod https://api.github.com/repos/actions/runner/releases; `
    $releaseTag = npx -q --yes semvermaxcli ( $releases | ? { ! $_.prerelease } | % { $_.tag_name -replace 'v(\d+\.\d+\.\d+)', '$1' }); `
    $assetsUrl = Invoke-RestMethod ($releases | ? { $_.tag_name -eq \"v$releaseTag\" }).assets_url; `
    $assetUrl = $assetsUrl | % { $_.browser_download_url } | ? { $_ -like \"*-win-x64-$releaseTag.zip\" }; `
    if ($assetUrl.count -ne 1) { throw \"Found more than one version.\" } `
    Invoke-WebRequest $assetUrl -OutFile C:\actions-runner.zip;

FROM mcr.microsoft.com/windows/servercore:ltsc${LTSC_YEAR}
ARG LTSC_YEAR
LABEL maintainer="giggio@giggio.net"
LABEL org.opencontainers.image.os=windows
LABEL org.opencontainers.image.os.version=10.0.20348
WORKDIR c:\actions-runner\
SHELL [ "powershell.exe", "-NoLogo", "-Command", "$ErrorActionPreference='Stop';", "$ProgressPreference='SilentlyContinue';" ]
RUN setx LTSC_YEAR $env:LTSC_YEAR
COPY --from=runner C:\actions-runner.zip C:\actions-runner\
RUN Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', '.'); `
    Remove-Item actions-runner.zip

# Install .NET Framework 4.8
RUN Invoke-WebRequest -Uri "https://download.visualstudio.microsoft.com/download/pr/2d6bb6b2-226a-4baa-bdec-798822606ff1/8494001c276a4b96804cde7829c04d7f/ndp48-x86-x64-allos-enu.exe" -OutFile "$env:TEMP\ndp48-installer.exe"; `
    Start-Process -FilePath "$env:TEMP\ndp48-installer.exe" -ArgumentList "/quiet", "/norestart" -Wait; `
    Remove-Item "$env:TEMP\ndp48-installer.exe"; `
    Start-Sleep -Seconds 30; `
    $dotnetVersion = (Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full' -ErrorAction SilentlyContinue).Version; `
    if (-not $dotnetVersion) { throw ".NET Framework 4.8 installation failed or not completed" }

# Install Chocolatey and packages
RUN [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12; `
    iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')); `
    Start-Sleep -Seconds 30; `
    # Add Chocolatey to PATH
    $chocoPath = 'C:\ProgramData\chocolatey\bin'; `
    $env:Path = "$env:Path;$chocoPath"; `
    [Environment]::SetEnvironmentVariable('Path', $env:Path, [EnvironmentVariableTarget]::Machine); `
    # Verify Chocolatey installation
    if (-not (Test-Path "$chocoPath\choco.exe")) { throw "Chocolatey installation failed" }; `
    # Install packages
    & "$chocoPath\choco.exe" install --no-progress -y git.install jq; `
    if ($LASTEXITCODE -ne 0) { throw "Chocolatey package installation failed" }; `
    # Refresh PATH after git installation
    $env:Path = [System.Environment]::GetEnvironmentVariable('Path', 'Machine')

# docker, index at: https://dockermsft.blob.core.windows.net/dockercontainer/DockerMsftIndex.json
RUN mkdir $env:temp\docker\extracted\ | out-null; `
    mkdir $env:ProgramFiles\Docker\ | out-null; `
    Invoke-WebRequest https://download.docker.com/win/static/stable/x86_64/docker-20.10.9.zip -OutFile $env:temp\docker\docker.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem ; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory(\"$env:temp\docker\docker.zip\", \"$env:temp\docker\extracted\"); `
    get-childitem -Recurse \"$env:temp\docker\extracted\docker\" | Move-Item -Destination \"$env:ProgramFiles\Docker\"; `
    Remove-Item $env:temp\docker\ -Recurse -Force; `
    [Environment]::SetEnvironmentVariable('PATH', \"$env:ProgramFiles\Docker\;$env:PATH\", [EnvironmentVariableTarget]::Machine); `
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; `
    Invoke-WebRequest https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Windows-x86_64.exe -UseBasicParsing -OutFile $Env:ProgramFiles\Docker\docker-compose.exe
COPY *.ps1 c:\actions-runner\
CMD ["powershell", "-f", "configureAndRun.ps1"]