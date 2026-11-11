# Android GSM/VoLTE ↔ SIP Gateway — Lab Guide

> **Independent homelab documentation (not an official Voice SIP Agent guide).**  
> Tested concept: Android phone with an active SIM card, Voice SIP Agent, Shizuku, Asterisk/Issabel and a SIP softphone.  
> All IP addresses and phone numbers below are **documentation examples**, not real environment details.

## Scope and results

This lab connects a cellular telephone line to an Asterisk PBX using an Android phone as a GSM/VoLTE-to-SIP bridge.

| Test | Lab result |
| --- | --- |
| Shizuku authorization | Passed |
| SIP registration | Passed |
| PBX extension → cellular outbound call | Passed |
| Cellular inbound call → PBX extension | Passed |
| Two-way audio | Passed after adjusting Android background permissions and audio mode |
| SIP inspection with sngrep | Passed |
| Long-term unattended operation | **Not validated** |
| Concurrent calls on a single handset | **Not supported in this lab** |

This is a **proof of concept**, not a production-ready carrier gateway.

## Components

- Android 11+ handset with SIM and Wi-Fi (device compatibility varies).
- [Voice SIP Agent](https://github.com/sparkcyf/android-voice-sip-agent) — SIP/cellular bridge.
- [Shizuku](https://shizuku.rikka.app/) — grants needed Android service access without rooting a supported device.
- Issabel/Asterisk with **PJSIP**.
- SIP desktop client, e.g. MicroSIP.
- sngrep for SIP signaling; tcpdump for RTP investigation.

## Lab topology

```text
                    Mobile network
                         |
                 Android + SIM
                 Voice SIP Agent
                192.0.2.71 (example)
                    SIP / RTP
                         |
                 Issabel / Asterisk
                192.0.2.10 (example)
                  /            \
          PJSIP gateway      SIP extension
               6000              1001
                                  |
                                Softphone
```

Use private, reachable network addressing in the real lab. The `192.0.2.0/24` addresses above are reserved for documentation (RFC 5737) and **must not be copied as functional LAN settings**.

## 1. Android preparation

1. Install Shizuku and start its service through Android wireless debugging, following its current official instructions.
2. On affected Realme/ColorOS devices, Shizuku may report restricted ADB permissions. A vendor-specific developer option named **Disable permission monitoring** may resolve the warning. This weakens a system protection: use only in a controlled lab and assess whether it can be restored afterward.
3. Install Voice SIP Agent from its project's official releases.
4. Authorize Voice SIP Agent in Shizuku and validate its **Health** indicators.
5. Grant appropriate telephone/call access, background execution, and unrestricted battery access as supported by the device.
6. Keep the SIP gateway active; Shizuku started via wireless debugging may need to be started again after device reboot.

Do not install unverified APKs or expose Shizuku access to untrusted apps.

## 2. Issabel test endpoint

For the **initial lab**, create a PJSIP extension for the Android gateway:

| Field | Example |
| --- | --- |
| Extension | `6000` |
| Display name | `ANDROID-GSM-LAB` |
| Secret | Generate a **unique strong password**, never commit it |
| Transport | UDP |
| Max contacts | 1 |
| Direct media | Off |
| Codec allow-list | PCMA (alaw), PCMU (ulaw) |
| Qualify frequency | 0 *only for troubleshooting* |

Confirm the actual configured PBX SIP port; it is not always 5060.

**Security:** A regular internal extension often inherits permissive dialing rights. Use a restricted context/CoS or a dedicated authenticated trunk for any extended deployment. Disabling `qualify` means registration/contact status is **not** an active SIP reachability probe.

## 3. Voice SIP Agent configuration

| Field | Example |
| --- | --- |
| SIP stack | `pjsua2` |
| Transport | `UDP` (LAN/VPN test) |
| PBX host | Your actual reachable PBX address |
| PBX port | Your actual PJSIP listening port |
| SIP domain | Empty, if the application uses PBX host as default |
| Extension/user | `6000` |
| Password | The strong secret configured in Issabel |
| Cellular inbound → SIP target | `1001` for a simple inbound test |
| SIM choice | Explicitly select the default outbound SIM |
| Allowed dial-out prefixes | Restrict for controlled tests |

Click **Save & apply**, then verify SIP registration and the Android permission health indicators.

## 4. Outbound call routing (PBX → SIM → mobile network)

A **Custom Trunk** can reference the PJSIP endpoint with this dial string:

```text
PJSIP/$OUTNUM$@6000
```

Create an **Outbound Route** exclusively for lab calls:

- Route name: `OUT-ANDROID-GSM-LAB`
- Dial pattern: prefix `87`, pattern `X.`, prepend empty
- Trunk sequence: the custom GSM trunk only
- Trunk maximum channels: `1`

For a number `<DDD><subscriber-number>`, dial:

```text
87<DDD><subscriber-number>
```

The PBX should remove `87` and emit a SIP INVITE whose Request-URI user is the **external destination number**, not the extension number `6000`.

**Caution:** `X.` is broad; it is convenient for isolated testing but unsafe as a production dialplan. Restrict prefixes, allowed numbers, extensions and cost exposure.

## 5. Inbound call routing (mobile network → SIM → PBX)

In Voice SIP Agent configure:

```text
Cellular inbound → SIP target = 1001
```

An incoming call to the SIM should cause an authenticated SIP INVITE for extension 1001. The PBX then rings the SIP client.

On Issabel check:

```asterisk
pjsip show endpoint 6000
pjsip show contacts
pjsip show endpoint 1001
```

A test endpoint using `from-internal` can route directly to the extension, but it must be hardened before production.

## 6. Diagnosing signaling with sngrep

Run from an authorized host that can observe both sides of the SIP traffic, e.g. the PBX:

```bash
sngrep -d any
sngrep -d any 'host <ANDROID_IP> or host <SOFTPHONE_IP>'
```

Replace placeholders with actual private IPs. Avoid posting raw captures publicly: SIP headers may include phone numbers, contact addresses and authentication material.

Typical successful call:

```text
INVITE → 100 Trying → 180 Ringing → 200 OK → ACK
                                                   ...RTP...
                                             BYE → 200 OK
```

An initial `401 Unauthorized` followed by a successfully authenticated INVITE is a normal SIP challenge, not necessarily a failure.

## 7. RTP and silent audio

A successful `200 OK` + `ACK` establishes the SIP session but **does not prove audible media**. Even RTP flowing both ways may carry silence.

Asterisk CLI:

```asterisk
rtp set debug on
rtp set debug off
```

When investigating a silent inbound call, verify the PBX receives and forwards packets between the gateway and the softphone, then verify the Android's audio bridge and actual sound.

A working combination observed during this lab was:

- `Cellular audio = Direct telephony`
- `Mute call audio on this phone = Off`
- Android permissions allowing the app to work in the background
- Consistent SIP contact / reachable address for the softphone

If the softphone registers a wrong virtual-interface IP, incoming calls may not ring. Correct its SIP bind/interface or NAT/contact handling and re-register. Do **not** assume a `180 Ringing` sent by the PBX proves that the softphone rang.

## 8. Troubleshooting summary

| Symptom | Check |
| --- | --- |
| Shizuku permission warning | App authorization, vendor permission monitoring restrictions |
| PJSIP contact `Unavailable` | SIP OPTIONS, firewall, contact address, Android background execution |
| Contact `NonQual` | Qualify disabled; **not proof** of reachability |
| Direct INVITE to `6000` gets 404 | Gateway expects a telephone destination in the Request-URI; dial through custom trunk/route |
| Inbound SIP reaches PBX, extension does not ring | Contact registered by softphone, virtual adapter/NAT address, INVITE response |
| Call rings and answers but is silent | RTP in both directions, codecs, Android audio mode, call background privileges |
| Only works while screen unlocked | Battery optimization, background call privileges, vendor process controls |
| After reboot no gateway | Shizuku service may require manual restart |
| Multiple simultaneous calls on one handset | Not supported by this single-handset test setup |

## 9. Privacy and security

This public document intentionally omits:

- Actual LAN/WAN IP addresses, hostnames, VPN endpoints
- Telephone numbers, caller ID values and customer details
- SIP credentials, authentication headers, API tokens and exported configuration JSON
- Real screenshots, packet captures and Asterisk logs

Do not commit `*.pcap`, SIP trace exports, Android app config exports, or credentials. Repository ownership and the GitHub account identity are public by design when the repository is public.

## 10. Limitations and next tests

- One active cellular call per gateway handset in this implementation/lab.
- Device-specific Android telephony/audio behavior; not every handset supports reliable capture.
- Reboot recovery, standby periods, long-duration calls, inbound/outbound overlap, failure recovery and carrier restrictions require more validation.
- For production, use VPN/private network connectivity, dedicated restricted SIP identities, monitoring and a supported multichannel GSM/VoLTE gateway where appropriate.
- Confirm carrier terms and applicable telecom rules before using a cellular plan as an outbound gateway.

## References

- [Voice SIP Agent — upstream repository](https://github.com/sparkcyf/android-voice-sip-agent)
- [Shizuku — official documentation](https://shizuku.rikka.app/)
- [Asterisk PJSIP documentation](https://docs.asterisk.org/Configuration/Channel-Drivers/SIP/Configuring-res_pjsip/)

---

**Status:** Documented lab test only. No executable app code is redistributed, and no endorsement by upstream authors is implied.
