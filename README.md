# PiVPN Google Authenticator Setup

Setting up Google Authenticator with [PiVPN](https://www.pivpn.io/) can be rather tricky.
I followed the instructions [here](https://github.com/pivpn/pivpn/issues/50#issuecomment-284082054) but had to change some of the configuration to get things working.

**This guide assumes that PiVPN is already set up.**

1. Install Google Authenticator.
   ```sh
   sudo apt-get install libpam-google-authenticator
   ```
1. Add the following lines to `/etc/openvpn/server.conf`:
   ```conf
   # Tells the OpenVPN server to validate the username/password
   # entered by clients using the authorization settings in the
   # 'openvpn' PAM module which we will create later.
   plugin /usr/lib/openvpn/openvpn-plugin-auth-pam.so openvpn
   # Disable renegotiation of data channel key. If enabled, the
   # VPN connection will be dropped when renegotiation takes
   # place because the OTP that was valid when the VPN connection
   # was established will no longer be valid.
   reneg-sec 0
   ```
1. Create the `openvpn` PAM module that we referenced in the step above, with the `common-account` PAM module as the base configuration.
   We create a separate PAM module so as to keep the configuration clean.
   ```sh
   cp /etc/pam.d/common-account /etc/pam.d/openvpn
   ```
1. Add the following line to `/etc/pam.d/openvpn`:
   ```conf
   auth required pam_google_authenticator.so user=root secret=/var/ga/${USER}/.google_authenticator
   ```
   We store the secret file in a non-home directory to ensure that the OpenVPN server is able to read the secret file.
   Additionally, we want the secret file to be readable only by `root` (400 permissions).
   Thus, we set the user to read the secret file to be `root`.
   More details can be found [here](https://github.com/google/google-authenticator-libpam#encrypted-home-directories).
1. Reload the configuration changes.
   ```sh
   sudo service openvpn restart
   ```

## Setting Up VPN User Accounts

Before setting up a VPN user account, you will need to determine the **username** and **password** that will be used to connect.

1. Create the directory to store the TOTP secret file for the user.
   **Replace _USERNAME_ below with the username of the user.**
   ```sh
   sudo mkdir /var/ga/USERNAME
   ```
1. Generate a time-based one-time password (TOTP) for the user.
   **Replace _USERNAME_ below with the username of the user.**
   ```sh
   sudo google-authenticator \
       `# Specify a non-standard file location` \
       --secret=/var/ga/USERNAME/.google_authenticator \
       `# Set up time-based (TOTP) verification` \
       --time-based \
       `# Disallow reuse of previously used TOTP tokens` \
       --disallow-reuse \
       `# Limit logins to 3 every 30 seconds` \
       --rate-limit=3 \
       --rate-time=30 \
       `# Set window of 3 permitted codes (one previous code, the current code, the next code)` \
       --window-size=3 \
       `# Don't confirm code; for interactive setups` \
       --no-confirm \
       `# Write file without first confirming with user` \
       --force
   ```
   The rationale for using a non-standard (non-home) file location is given in the instructions above.
1. Note down the secret key.
   This will be used by the user to set up their TOTP in Google Authenticator.
1. Create the VPN user account.
   **Replace _USERNAME_ and _PASSWORD_ below with the username and password of the user respectively.**
   ```sh
   pivpn add --name USERNAME --password PASSWORD --days 1080
   ```
1. Add the following lines to `USERNAME.ovpn`:
   ```
   # Request username and password when connecting.
   auth-user-pass
   # Disable renegotiation of data channel key. If enabled, the
   # VPN connection will be dropped when renegotiation takes
   # place because the OTP that was valid when the VPN connection
   # was established will no longer be valid.
   reneg-sec 0
   ```
   Comment out the following line in `USERNAME.ovpn`:
   ```
   # Do not cache passwords in memory as the password is actually a TOTP.
   # auth-nocache
   ```
1. Download the generated `USERNAME.ovpn` file for the user.
