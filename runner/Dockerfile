# escape=`
ARG LTSC_YEAR=2019
FROM mcr.microsoft.com/windows/servercore:ltsc${LTSC_YEAR} as runner

SHELL [ "powershell.exe", "-NoLogo", "-Command", "$ErrorActionPreference='Stop';", "$ProgressPreference='SilentlyContinue';" ]

# runner, auto download latest release
RUN Invoke-WebRequest https://nodejs.org/dist/v16.15.1/node-v16.15.1-win-x64.zip -OutFile node.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory("$PSScriptRoot\node.zip", "$env:TEMP\node"); `
    Move-Item "$env:TEMP\node\node-v16.15.1-win-x64" c:\node\; `
    $env:Path="c:\node\;$env:PATH"; `
    $releases = Invoke-RestMethod https://api.github.com/repos/actions/runner/releases; `
    $releaseTag = npx -q --yes semvermaxcli ( $releases | Where-Object { -not $_.prerelease } | ForEach-Object { $_.tag_name -replace 'v(\d+\.\d+\.\d+)', '$1' }); `
    $assetsUrl = Invoke-RestMethod ($releases | Where-Object { $_.tag_name -eq "v$releaseTag" }).assets_url; `
    $assetUrl = $assetsUrl | ForEach-Object { $_.browser_download_url } | Where-Object { $_ -like "*-win-x64-$releaseTag.zip" }; `
    if ($assetUrl.Count -ne 1) { throw "Found more than one version."; } `
    Invoke-WebRequest $assetUrl -OutFile C:\actions-runner.zip;

FROM mcr.microsoft.com/windows/servercore:ltsc${LTSC_YEAR}
ARG LTSC_YEAR

LABEL maintainer="giggio@giggio.net"
WORKDIR c:\actions-runner\

SHELL [ "powershell.exe", "-NoLogo", "-Command", "$ErrorActionPreference='Stop';", "$ProgressPreference='SilentlyContinue';" ]

RUN setx LTSC_YEAR $env:LTSC_YEAR

COPY --from=runner C:\actions-runner.zip C:\actions-runner\

RUN Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', '.'); `
    Remove-Item actions-runner.zip

# Install Chocolatey and packages git, jq
RUN powershell.exe -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "[System.Net.ServicePointManager]::SecurityProtocol = 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))"
RUN choco install --no-progress -y git.install jq

RUN setx PATH "$env:PATH;C:\Program Files\Git\usr\bin\"

# Install Docker CLI from Microsoft stable channel (updated URL)
RUN mkdir $env:TEMP\docker\extracted\ | Out-Null; `
    mkdir $env:ProgramFiles\Docker\ | Out-Null; `
    Invoke-WebRequest https://dockermsft.azureedge.net/dockercontainer/docker-20-10-9.zip -OutFile $env:TEMP\docker\docker.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory("$env:TEMP\docker\docker.zip", "$env:TEMP\docker\extracted\"); `
    Get-ChildItem -Recurse "$env:TEMP\docker\extracted\docker" | Move-Item -Destination "$env:ProgramFiles\Docker"; `
    Remove-Item $env:TEMP\docker\ -Recurse -Force; `
    [Environment]::SetEnvironmentVariable('PATH', "$env:ProgramFiles\Docker;$env:PATH", [EnvironmentVariableTarget]::Machine); `
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; `
    Invoke-WebRequest https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Windows-x86_64.exe -UseBasicParsing -OutFile $Env:ProgramFiles\Docker\docker-compose.exe

COPY *.ps1 c:\actions-runner\

CMD ["powershell", "-f", "configureAndRun.ps1"]