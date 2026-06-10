  FRAPPE / ERPNEXT VERSION 16 - WINDOWS WSL INSTALLATION GUIDE
  Ubuntu 24.04 LTS | Frappe v16 | ERPNext v16

NOTE: Replace [frappe-user] with your actual Linux username
      and [site-name] with your desired site name throughout.

----------------------------------------------------------------
PART 1 - WSL SETUP
----------------------------------------------------------------

Step 1: Install WSL
-------------------
Open PowerShell as Administrator and run:

    wsl --install

After installation, you will be prompted to set a Linux
username and password. If not prompted immediately, restart
your PC — it will appear on next boot.

Step 2: Verify WSL Version
--------------------------
Open PowerShell and confirm you are running WSL2:

    wsl --list --verbose

The VERSION column should show 2. If it shows 1, upgrade with:

    wsl --set-version Ubuntu 2

Step 3: (Optional) Windows Terminal
------------------------------------
Install Windows Terminal from the Microsoft Store.
In Settings > Startup, set the default profile to Ubuntu
(the penguin icon).


----------------------------------------------------------------
PART 2 - FRAPPE INSTALLATION
----------------------------------------------------------------

2.1  Update and Upgrade Packages
---------------------------------
    sudo apt-get update -y
    sudo apt-get upgrade -y


2.2  Set User Permissions
--------------------------
    sudo usermod -aG sudo [frappe-user]
    sudo su [frappe-user]
    cd /home/[frappe-user]


2.3  Install Git
-----------------
    sudo apt-get install git -y


2.4  Install Python Dependencies
----------------------------------
NOTE: Ubuntu 24.04 ships with Python 3.12. Frappe v16
      requires Python 3.11 or higher.

    sudo apt-get install python3-dev python3-setuptools python3-pip -y
    sudo apt install python3.12-venv -y


2.5  Install MariaDB
---------------------
    sudo apt-get install software-properties-common -y
    sudo apt install mariadb-server mariadb-client -y
    sudo mysql_secure_installation

When running mysql_secure_installation, answer as follows:

    Prompt                              Answer
    ----------------------------------  --------------------------------
    Enter current password for root     Press Enter (no password yet)
    Switch to unix_socket auth?         Y
    Change the root password?           Y  (set a strong password)
    Remove anonymous users?             Y
    Disallow root login remotely?       N  (allows BI tools like Metabase)
    Remove test database?               Y
    Reload privilege tables?            Y

Configure MariaDB character set:

    sudo nano /etc/mysql/my.cnf

Append the following block at the end of the file:

    [mysqld]
    character-set-client-handshake = FALSE
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci

    [mysql]
    default-character-set = utf8mb4

Restart MariaDB:

    sudo service mysql restart


2.6  Install Redis Server
--------------------------
    sudo apt-get install redis-server -y


2.7  Install Node.js (via NVM), NPM, and Yarn
-----------------------------------------------
NOTE: Frappe v16 supports Node 18 LTS. Do not use Node 20
      or Node 22 unless Frappe release notes confirm support.

    sudo apt install curl -y
    curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
    source ~/.profile
    nvm install 18
    sudo apt-get install npm -y
    sudo npm install -g yarn


2.8  Install wkhtmltopdf
-------------------------
NOTE: Required for PDF print format generation.

    sudo apt-get install xvfb libfontconfig wkhtmltopdf -y


2.9  Install Frappe Bench and Ansible
---------------------------------------
    sudo -H pip3 install frappe-bench --break-system-packages
    sudo -H pip3 install ansible --break-system-packages


----------------------------------------------------------------
PART 3 - INITIALIZE FRAPPE BENCH
----------------------------------------------------------------

3.1  Initialize Bench for Version 16
--------------------------------------
    bench init frappe-bench --frappe-branch version-16
    cd frappe-bench

NOTE: This downloads Frappe from GitHub and sets up a Python
      virtual environment. May take several minutes.


3.2  Fix Directory Permissions
--------------------------------
    chmod -R o+rx /home/[frappe-user]


3.3  Create a New Site
-----------------------
    bench new-site [site-name]

You will be prompted for:
  - MariaDB root password (set during mysql_secure_installation)
  - Administrator password for the new site


----------------------------------------------------------------
PART 4 - INSTALL ERPNEXT AND APPS
----------------------------------------------------------------

4.1  Get ERPNext and Optional Apps
------------------------------------
    bench get-app --branch version-16 payments
    bench get-app --branch version-16 erpnext
    bench get-app --branch version-16 hrms


4.2  Install Apps on Your Site
--------------------------------
    bench --site [site-name] install-app erpnext
    bench --site [site-name] install-app hrms
    bench migrate


----------------------------------------------------------------
PART 5 - PRODUCTION SETUP
----------------------------------------------------------------

5.1  Enable Scheduler and Disable Maintenance Mode
----------------------------------------------------
    bench --site [site-name] enable-scheduler
    bench --site [site-name] set-maintenance-mode off


5.2  Test Run
--------------
    bench start

Visit http://localhost:8000 in your browser. If the site
loads correctly, press Ctrl+C and proceed.


5.3  Setup Production Services
--------------------------------
    sudo bench setup production [frappe-user]


5.4  Setup Nginx
-----------------
    bench setup nginx

NOTE: A harmless traceback after Ctrl+C interrupt is normal.
      Proceed to the next step.


5.5  Verify and Restart All Services
--------------------------------------
    bench start

If bench start does not terminate on its own, run:

    sudo bench setup production [frappe-user]
    sudo supervisorctl restart all
    sudo bench setup production [frappe-user]


----------------------------------------------------------------
PART 6 - DEVELOPER MODE (OPTIONAL)
----------------------------------------------------------------

Enable developer mode if using Frappe for development:

    bench --site [site-name] set-config developer_mode 1

WARNING: Do NOT enable developer mode in a production environment.
         It enables live JS/CSS reloading and exposes app source.


----------------------------------------------------------------
PART 7 - TROUBLESHOOTING AND TIPS
----------------------------------------------------------------

Common Issues
--------------

  Issue                            Solution
  -------------------------------  ----------------------------------------
  pip install fails                Always use --break-system-packages flag
  bench init hangs at Node         Ensure: nvm install 18 completed
  MariaDB connection refused       Run: sudo service mysql start
  Redis connection error           Run: sudo service redis-server start
  Port 8000 already in use         Run: pkill -f "bench start"
  npm/yarn command not found       Run: source ~/.profile after nvm install
  Supervisor not restarting        Run: sudo supervisorctl reread
                                        sudo supervisorctl update


Useful Service Commands
------------------------

    # Start services
    sudo service mysql start
    sudo service redis-server start

    # Restart all supervisor workers
    sudo supervisorctl restart all

    # Bench update (after going live)
    bench update --reset

    # Check bench logs
    cd frappe-bench
    tail -f logs/web.error.log
    tail -f logs/worker.error.log


----------------------------------------------------------------
  Frappe / ERPNext Version 16 - WSL Installation Guide
