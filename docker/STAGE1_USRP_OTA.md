# Stage 1 USRP OTA validation

This runbook defines the first over-the-air validation of the blind CFO-PSS
Stage 1 receiver on the two laboratory N310s. It is deliberately limited to a
1x1 PHY-test experiment, so the 5G Core is not required for the initial
synchronization and PBCH validation.

## Hardware assignment

| Role | Host | N310 serial | Management/data address |
|---|---|---|---|
| gNB | `pws-03` | `32B1A33` | `192.168.20.2` |
| nrUE | `pws-02` | `3249D76` | `192.168.20.2` |

The identical N310 addresses are valid because each USRP is on a separate
private Ethernet link to its host. Both containers use host networking.

## Radio profile

The profile is derived from OAI's
`gnb.sa.band78.106prb.n310.7ds2u.conf` example:

- NR band n78;
- carrier center approximately 3.42501 GHz;
- SSB approximately 3.4248 GHz;
- numerology 1 (30 kHz SCS);
- 106 PRBs (approximately 40 MHz);
- one TX and one RX channel on subdevice `A:0`;
- internal clock and internal time source;
- 20 MHz UHD tuning offset;
- N310/OAI timing advance `-A 90`.

The gNB configuration is
`ci-scripts/conf_files/gnb.sa.band78.106prb.n310.tfm-stage1-ota-1x1.conf`.
The stock OAI file is not modified.

## Conservative gain interpretation

OAI passes `att_tx` to the USRP driver as attenuation from the device's maximum
TX gain. The N310 reports a maximum TX gain of 65 dB, so `att_tx = 55` requests
approximately 10 dB of hardware TX gain. On RX, `max_rxgain = 75` and
`att_rx = 35` request 40 dB.

The nrUE command follows the same driver convention: `--ue-txgain 55` requests
approximately 10 dB of N310 TX gain, while `--ue-rxgain 40` requests 40 dB RX
gain. Confirm the applied values in the UHD/OAI startup logs. Increase RF power
only after evaluating the received level; never start from the stock
`att_tx = 0` value in a same-room OTA setup.

## Safety boundary

Do not run either softmodem until use of band n78 around 3.425 GHz has been
authorized for this laboratory. The commands below tune and operate the N310s
and the gNB command transmits RF. They do not update N310 firmware, FPGA, MPM,
EEPROM, or network configuration.

## Preflight

The following conditions must hold on both hosts:

- branch `tfm/stage1-usrp-validation` is up to date;
- image `tfm-stage1-usrp-build:uhd4.4` exists;
- UHD 4.4 is reported inside the image;
- all five `test_nr_pss_cfo_*` tests pass;
- `uhd_find_devices` reports the expected serial and `claimed: False`;
- no other user has reserved either N310.

## First OTA pilot

Once RF use is authorized, start the gNB on `pws-03` first. The bind mount is
required because the already-built development image predates this new config.

```bash
sudo docker run --rm --name tfm-stage1-gnb --network host --cap-add SYS_NICE --cap-add IPC_LOCK --ulimit memlock=-1:-1 -v /home/tim/NTN_enrique/openairinterface5g/ci-scripts/conf_files/gnb.sa.band78.106prb.n310.tfm-stage1-ota-1x1.conf:/oai-ran/ci-scripts/conf_files/gnb.sa.band78.106prb.n310.tfm-stage1-ota-1x1.conf:ro tfm-stage1-usrp-build:uhd4.4 /oai-ran/cmake_targets/ran_build/build/nr-softmodem -O /oai-ran/ci-scripts/conf_files/gnb.sa.band78.106prb.n310.tfm-stage1-ota-1x1.conf --phy-test --tune-offset 20000000 -A 90 --log_config.global_log_options level,nocolor,time
```

Before starting the UE, verify in the gNB log that the selected device serial is
`32B1A33`, TX gain is approximately 10 dB, RX gain is approximately 40 dB, and
streaming starts without late/underflow errors.

Start the Stage 1 nrUE on `pws-02`:

```bash
sudo docker run --rm --name tfm-stage1-nrue --network host --cap-add SYS_NICE --cap-add IPC_LOCK --ulimit memlock=-1:-1 tfm-stage1-usrp-build:uhd4.4 /oai-ran/cmake_targets/ran_build/build/nr-uesoftmodem --phy-test -O /oai-ran/ci-scripts/conf_files/ue.sa.conf --usrp-args "type=n3xx,serial=3249D76,addr=192.168.20.2,clock_source=internal,time_source=internal" --tx_subdev A:0 --rx_subdev A:0 -r 106 --numerology 1 --band 78 -C 3425010000 --ssb 509 --initial-fo 0 --ue-fo-compensation --ue-pss-cfo-search --ue-rxgain 40 --ue-txgain 55 --tune-offset 20000000 -A 90 --log_config.global_log_options level,nocolor,time
```

Success requires more than a PSS candidate. Record the Stage 1 CFO/NID2 result,
SSS success, PBCH success and delivery of the decoded MIB/PDU to the upper PHY
path. Stop both processes with `Ctrl+C` after collecting the evidence.

## Planned A/B sequence

After the zero-offset pilot is stable, use the same gNB run for three UE cases:

1. original receiver at the nominal carrier, as the hardware sanity control;
2. original receiver with a controlled receiver tuning mismatch and Stage 1
   disabled, as the negative control;
3. Stage 1 receiver with the same mismatch, as the treatment case.

Start with a 50 kHz receive tuning mismatch, which remains inside Stage 1's
default +/-60 kHz coarse span. Test both signs only after the nominal pilot.
Do not change the gNB frequency to create the CFO: change only the UE receiver
carrier argument so that the transmitted RF profile remains fixed.
