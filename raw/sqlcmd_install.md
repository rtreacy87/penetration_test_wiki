sqlcmd is part of Microsoft's SQL Server tools, and Kali's default repositories usually do not include it. You need to add Microsoft's repo first.

Since you're on Kali (Debian-based), do this:

1. Install prerequisites
sudo apt update
sudo apt install -y curl gnupg2 apt-transport-https ca-certificates
2. Add Microsoft's signing key
curl https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
3. Add the Microsoft repository

For Kali/Debian 12-compatible systems:

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft-prod.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" | sudo tee /etc/apt/sources.list.d/mssql-release.list
4. Update package lists
sudo apt update
5. Install sqlcmd

Newer versions package it as mssql-tools18:

sudo ACCEPT_EULA=Y apt install -y mssql-tools18 unixodbc-dev
6. Add sqlcmd to PATH
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.zshrc
source ~/.zshrc

(If you're using bash instead of zsh, use ~/.bashrc.)

7. Verify installation
sqlcmd -?

You should now see the help output.
