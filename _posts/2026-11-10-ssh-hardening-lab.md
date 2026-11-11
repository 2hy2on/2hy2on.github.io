---
title: "Ubuntu SSH 2FA Hardening Lab"
date: 2026-11-10 17:30:00 +0000
categories: [Security, Hands-on Lab]
tags: [ssh, 2fa, ubuntu, hardening]
---

## 🎯 Summary
This lab hardens SSH authentication on an Ubuntu 22.04 LTS server by adding multi-factor authentication (MFA/2FA) with Google Authenticator.

---

## 🖥️ Environment

| Item | Details |
| --- | --- |
| OS | Ubuntu 22.04 LTS (VMware / VirtualBox) |
| MFA provider | Google Authenticator (`pam_google_authenticator`) |
| Client | Local terminal / SSH client |


---

## 🛠️ Steps

### 1. Installing Google Authenticator PAM Module
First, update the package repository and install the PAM module:
``` bash
sudo apt update && sudo apt install libpam-google-authenticator -y
```
### 2. Generate the 2FA keys
Run the setup tool to generate the secret key and QR code:

``` bash
google-authenticator
```

- scan the generated QR code with the Google Authenticator app.
- Save the emergency scratch codes securely.
- select **y** to time-based tokens and to the rate-limiting options.

### 3. Configure PAM (/etc/pam.d/sshd)
Edit the SSH PAM configuration to enforce OTP authentication
- Add the following line to requirenment rules:

```bash
auth required pam_google_authenticator.so 
```

### 4. Configure the SSH daemon (/etc/ssh/sshd_config)

Modify the SSH daemon settings to enforce a combination of **Public Key** and **Interactive (OTP)** authentication:

Restart the SSH daemon to apply changes, and upon attempting to connect, you will be prompted for your password and the verification code:

<div style="text-align: center;">
  <img src="/assets/lib/img/posts/ssh-login-success.png" style="width: 65%; border-radius: 10px; box-shadow: 0 4px 10px rgba(0,0,0,0.3); display: block; margin: 0 auto;" alt="SSH 2FA Login Verification">
  <p style="font-style: italic; font-size: 0.9rem; color: #666; margin-top: 8px;">Figure: Successful SSH login prompt requiring password and verification code (OTP)</p>
</div>

``` bash
KbdInteractiveAuthentication yes
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive
```

## 🚨 Key Takeaways & Security Reflection

### 💡Authentication Model: Public Key + OTP vs. Password + OTP
Rather than stacking OTP on top of a standard account password, this implemetation strictly pairs SSH Public Key Authentication with `keyboard-interactive` TOTP tokens:

- **Password + OTP**: While implementing rate-limiting (such as locking out after 3 failed attempts) helps mitigate brute-force attacks, it still relies on static credentials that can be phished, leaked, or intercepted.

- **Public Key + OTP**: Disables password authentication entirely. Attackers cannot even reach the OTP challenge prompt without presenting a valid cryptographic private key, effectively neutralizing brute-force and credential-stuffing vectors.

### 💡 Network Adapter Risks: NAT vs. Bridged Mode
During this lab, switching the virtual machine's network adapter from NAT to Bridged Mode highlighted a critical infrastructure security consideration.

| Mode | What it means | Risk |
| --- | --- | --- |
| **NAT** | The VM hides behind computer, Other devices cannot see it. | Low. Outsider cannot reach it directly. |
| **Bridged** | The VM gets its own IP from the local network's DHCP server. | Higher. Anyone on the same Wi-Fi can try to log in to SSH (port 22)|


On an untrusted network such as public Wi-Fi at a cafe or university, Bridged mode widens the attack surface considerably.

### Conclusion
- Keep lab VMs in NAT or Host-Only mode when working on public networks.
- If Bridged mode is required, enable strict UFW rules and enforce key-based login with 2FA before connecting.
