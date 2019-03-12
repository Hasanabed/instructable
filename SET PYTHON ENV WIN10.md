# SET UP PYTHON ENVIRONMENT WIN10

# Open powershell in admin mode:

## INSTALL CHOCOLATEY


Set-ExecutionPolicy AllSigned

Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))


## INSTALL PYTHON

choco install python3

## INSTAL ATOM

choco install atom

## INSTALL GIT

choco install git -y /GitAndUnixToolsOnPath

## INSTALL AWSCLI

choco install awscli

## INSTALL OPENSSH

choco install openssh -y -params "/SSHAgentFeature"

## INSTAL iPython

pip install ipython

## check if python is intalled
python -V
