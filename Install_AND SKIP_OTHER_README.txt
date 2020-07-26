Of course I have made a bugfix, including some other small fixes, and created a pull request. The bad news is, there were already 5 pull request waiting, the oldest waiting for more than a year (and coincidentally for the very same bug, just a different solution). I wouldn’t be surprised if my pull request will be ignored as well.

So, what can we do to install Docker EE on Windows 10? Well, I have two options for you. Option number 1 is do a manual install, which is quite easy to do. Option number 2 is to download my version of the module DockerMsftProvider and let it install Docker for you.

Option 1: Manual install
The documentation of Docker EE contains a step-by-step instruction to use a script to install Docker EE. Follow that script and you will be safe. It can also be used to update Docker, just by downloading the latest files and overwrite the existing files.

Here is a modified version of that script. It will automatically detect the latest stable version, download and extract it and install it as a service.

# Install Windows feature containers
$restartNeeded = $false
if (!(Get-WindowsOptionalFeature -FeatureName containers -Online).State -eq 'Enabled') {
    $restartNeeded = (Enable-WindowsOptionalFeature -FeatureName containers -Online).RestartNeeded
}

if (Get-Service docker -ErrorAction SilentlyContinue)
{
    Stop-Service docker
}

# Download the zip file.
$json = Invoke-WebRequest https://download.docker.com/components/engine/windows-server/index.json | ConvertFrom-Json
$stableversion = $json.channels.stable.alias
$version = $json.channels.$stableversion.version
$url = $json.versions.$version.url
$zipfile = Join-Path "$env:USERPROFILE\Downloads\" $json.versions.$version.url.Split('/')[-1]
Invoke-WebRequest -UseBasicparsing -Outfile $zipfile -Uri $url

# Extract the archive.
Expand-Archive $zipfile -DestinationPath $Env:ProgramFiles -Force

# Modify PATH to persist across sessions.
$newPath = [Environment]::GetEnvironmentVariable("PATH",[EnvironmentVariableTarget]::Machine) + ";$env:ProgramFiles\docker"
$splittedPath = $newPath -split ';'
$cleanedPath = $splittedPath | Sort-Object -Unique
$newPath = $cleanedPath -join ';'
[Environment]::SetEnvironmentVariable("PATH", $newPath, [EnvironmentVariableTarget]::Machine)
$env:path = $newPath

# Register the Docker daemon as a service.
if (!(Get-Service docker -ErrorAction SilentlyContinue)) {
  dockerd --exec-opt isolation=process --register-service
}

# Start the Docker service.
if ($restartNeeded) {
    Write-Host 'A restart is needed to finish the installation' -ForegroundColor Green
    If ((Read-Host 'Do you want to restart now? [Y/N]') -eq 'Y') {
      Restart-Computer
    }
} else {
    Start-Service docker
}


Run this script, and you will have Docker EE installed on Windows 10. Make sure to save the script and use it again to update to a newer version.


Portainer
Another benefit of installing Docker EE is that Portainer works out of the box. You don’t need to do any settings, like exposing port 2375. The next two docker commands download and install Portainer for you.

docker pull portainer/portainer
docker run -d --restart always --name portainer --isolation process -h portainer -p 9000:9000 -v //./pipe/docker_engine://./pipe/docker_engine portainer/portainer

Open portainer with http://localhost:9000, provide a username and password and then click on Manage the local Docker environment. Click Connect to continue.


