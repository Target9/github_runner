# escape=`
ARG RUNNER_NAME
ARG WINDOWS_VERSION=ltsc2022
FROM mcr.microsoft.com/windows/servercore:${WINDOWS_VERSION} as runner
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

FROM mcr.microsoft.com/windows/servercore:${WINDOWS_VERSION}
ARG RUNNER_NAME 
LABEL maintainer="giggio@giggio.net"
LABEL org.opencontainers.image.os=windows
LABEL org.opencontainers.image.os.version=10.0.20348
WORKDIR c:\actions-runner\
SHELL [ "powershell.exe", "-NoLogo", "-Command", "$ErrorActionPreference='Stop';", "$ProgressPreference='SilentlyContinue';" ]
RUN if ($env:RUNNER_NAME) { setx RUNNER_NAME $env:RUNNER_NAME }
COPY --from=runner C:\actions-runner.zip C:\actions-runner\
RUN Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', '.'); `
    Remove-Item actions-runner.zip

# Install .NET Framework 4.8
RUN $dotnet48Url = "https://download.visualstudio.microsoft.com/download/pr/2d6bb6b2-226a-4baa-bdec-798822606ff1/8494001c276a4b96804cde7829c04d7f/ndp48-x86-x64-allos-enu.exe"; `
    $dotnet48Installer = "$env:TEMP\ndp48-installer.exe"; `
    Write-Host "Downloading .NET Framework 4.8..."; `
    Invoke-WebRequest -Uri $dotnet48Url -OutFile $dotnet48Installer; `
    Write-Host "Installing .NET Framework 4.8..."; `
    Start-Process -FilePath $dotnet48Installer -ArgumentList "/quiet", "/norestart" -Wait -PassThru; `
    Write-Host "Cleaning up .NET Framework 4.8 installer..."; `
    Remove-Item $dotnet48Installer -Force; `
    Write-Host ".NET Framework 4.8 installation completed."

# Install Chocolatey and required packages
RUN [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12; `
    Write-Host "Installing Chocolatey..."; `
    iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')); `
    $env:Path = "$env:Path;C:\ProgramData\chocolatey\bin"; `
    Write-Host "Configuring Chocolatey..."; `
    choco feature enable -n allowGlobalConfirmation; `
    Write-Host "Installing Git and jq..."; `
    choco install --no-progress -y git.install jq

# Set up Git path
RUN setx PATH \"$env:PATH;C:\Program Files\Git\usr\bin\"

# Install Docker
RUN Write-Host "Installing Docker..."; `
    mkdir $env:temp\docker\extracted\ | out-null; `
    mkdir $env:ProgramFiles\Docker\ | out-null; `
    Invoke-WebRequest https://download.docker.com/win/static/stable/x86_64/docker-20.10.9.zip -OutFile $env:temp\docker\docker.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem ; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory(\"$env:temp\docker\docker.zip\", \"$env:temp\docker\extracted\"); `
    get-childitem -Recurse \"$env:temp\docker\extracted\docker\" | Move-Item -Destination \"$env:ProgramFiles\Docker\"; `
    Remove-Item $env:temp\docker\ -Recurse -Force; `
    [Environment]::SetEnvironmentVariable('PATH', \"$env:ProgramFiles\Docker\;$env:PATH\", [EnvironmentVariableTarget]::Machine); `
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; `
    Write-Host "Installing Docker Compose..."; `
    Invoke-WebRequest https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Windows-x86_64.exe -UseBasicParsing -OutFile $Env:ProgramFiles\Docker\docker-compose.exe

COPY *.ps1 c:\actions-runner\
CMD ["powershell", "-f", "configureAndRun.ps1"]
