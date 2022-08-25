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
