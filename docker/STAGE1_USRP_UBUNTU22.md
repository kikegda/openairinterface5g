<!-- SPDX-License-Identifier: CC-BY-4.0 -->

# Stage 1 USRP build environment

This development image builds the TFM Stage 1 receiver for the two shared
N310 hosts without changing their Ubuntu 20.04 userspace. It deliberately
uses:

- Ubuntu 22.04 inside the container;
- UHD 4.4.0.0, matching the existing N310 MPM/FPGA installation;
- the current `tfm/stage1-usrp-validation` source tree;
- only the NR gNB, NR UE, and Stage 1 unit tests.

The Docker build compiles UHD and OAI but does not execute
`uhd_find_devices`, `uhd_usrp_probe`, `uhd_image_loader`, or either OAI
softmodem. Therefore, building the image does not discover, initialize,
configure, or update a USRP.

## Build

Run from the OAI repository root:

```bash
sudo docker build \
  --target stage1-usrp-build \
  --file docker/Dockerfile.stage1-usrp.ubuntu22 \
  --tag tfm-stage1-usrp-build:uhd4.4 \
  .
```

The first build downloads and compiles UHD and OAI and can take a substantial
amount of time. Do not run `docker system prune` on the shared hosts.

## Offline verification

These commands inspect the completed image and do not give it host networking
or USRP device arguments:

```bash
sudo docker run --rm tfm-stage1-usrp-build:uhd4.4 \
  uhd_config_info --version
```

```bash
sudo docker run --rm tfm-stage1-usrp-build:uhd4.4 \
  bash -lc "cd /oai-ran/cmake_targets/ran_build/build && \
  ctest --output-on-failure -R '^test_nr_pss_cfo_'"
```

The expected UHD major/minor version is `4.4`, and all five Stage 1 tests must
pass.

## Hardware boundary

Building and performing the offline verification are separate from hardware
execution. A later, explicitly approved runbook will define host networking,
real-time permissions, N310 addresses, RF ports, gains, frequencies, and the
physical attenuation setup. Do not add `--network host`, `--privileged`, or
USRP device arguments during this build-only step.
