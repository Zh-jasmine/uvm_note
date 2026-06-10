# AXI-LITE2SPI Verification Plan

## 0. Status

| Item | Status |
|---|---|
| DUT | AXI4-Lite Slave -> SPI Master bridge |
| Scope | AXI-Lite register/protocol + SPI function/timing + reset + RAL |
| Regression | 20 tests in `UVMTB/sim/Makefile` |
| Main result | Functional coverage 100%, SVA coverage 100%, DUT line/branch about 90% with waivers |
| Remaining | Read/write concurrency is optional; not a blocking item for current project scope |

Legend: `DONE` completed and covered by regression; `WAIVE` accepted by design intent; `TODO` optional/future work.

---

## 1. Coverage Matrix

| Area | Checkpoint | Status | Test / Implementation |
|---|---|---:|---|
| AXI-Lite | Register frontdoor write | DONE | Common config sequences + directed tests |
| AXI-Lite | Readable registers (`busy`, `miso_data`) | DONE | `axi_busy_test`, MISO tests, `ral_test` RO read |
| AXI-Lite | Write-only register behavior | DONE | RAL frontdoor write + backdoor peek |
| AXI-Lite | AW/W handshake timing | DONE | `axi_handshake_test`: simultaneous, AW first, W first, long-delay cases |
| AXI-Lite | WSTRB byte select | DONE | `axi_wstrb_test`: byte lane + sparse mask |
| AXI-Lite | Busy/status | DONE | `axi_busy_test`: busy during transfer, idle after transfer |
| AXI-Lite | BRESP/RRESP OKAY path | DONE | Driver captures response; DUT fixed OKAY |
| AXI-Lite | Illegal address write/read | WAIVE | Default keeps value / returns 0; not a key project target |
| AXI-Lite | Read/write concurrent transactions | TODO | Optional protocol stress; current driver is transaction-serial |
| SPI | Mode 0/1/2/3 | DONE | `spi_mode_test`, `spi_random_test` |
| SPI | Word length 32/16/8/4 | DONE | `spi_word_len_test`, `spi_random_test` |
| SPI | SCK speed 4 settings | DONE | `spi_sck_speed_test`, SVA |
| SPI | MOSI data patterns | DONE | all-0, all-1, alternating, random |
| SPI | MISO readback | DONE | `spi_miso_read_test`, all-0, all-1 |
| SPI | Back-to-back frames | DONE | `spi_burst_test` |
| SPI | Random regression | DONE | `spi_random_test`: 80 constrained frames |
| SPI timing | CS->SCK delay | DONE | `spi_cs_sck_test`, `CS_SCK_CHK`, `CS_SCK_COV` |
| SPI timing | SCK period | DONE | `spi_sck_speed_test`, `SCK_SPEED_CHK`, `SCK_SPEED_COV` |
| SPI timing | SCK->CS delay | DONE | `spi_sck_cs_test`, `SCK_CS_CHK` |
| SPI timing | IFG inter-frame gap | DONE | `spi_ifg_test` |
| Reset | AXI idle during reset | DONE | `axi_reset_sva.sv` |
| Reset | SPI pins idle during reset | DONE | `spi_reset_sva.sv` |
| Reset | Reset during transfer and recovery | DONE | `reset_mid_test`, `reset_sva_test` |
| RAL | Register model + adapter | DONE | `axi_spi_reg_block.sv`, `axi_spi_reg_adapter.sv` |
| RAL | Frontdoor/backdoor smoke | DONE | `ral_test` |

---

## 2. Regression Tests

| # | Test | Purpose |
|---:|---|---|
| 1 | `spi_word_len_test` | 32/16/8/4-bit word length |
| 2 | `spi_mode_test` | SPI mode 0/1/2/3 |
| 3 | `spi_cs_sck_test` | CS-to-SCK delay |
| 4 | `spi_sck_cs_test` | SCK-to-CS delay |
| 5 | `spi_ifg_test` | Inter-frame gap |
| 6 | `spi_sck_speed_test` | SCK divider / period |
| 7 | `axi_busy_test` | Busy/status behavior |
| 8 | `axi_handshake_test` | AXI AW/W timing variants |
| 9 | `axi_wstrb_test` | WSTRB byte lane select |
| 10 | `spi_burst_test` | Back-to-back SPI frames |
| 11 | `spi_alternating_test` | 0x55 / 0xAA MOSI pattern |
| 12 | `spi_data_all_0_test` | MOSI all-zero data |
| 13 | `spi_data_all_1_test` | MOSI all-one data |
| 14 | `reset_mid_test` | Reset during active transfer |
| 15 | `spi_miso_read_test` | MISO readback 0x5A |
| 16 | `spi_miso_all_0_test` | MISO all-zero |
| 17 | `spi_miso_all_1_test` | MISO all-one |
| 18 | `spi_random_test` | 80 constrained random frames |
| 19 | `reset_sva_test` | Reset SVA multi-scenario test |
| 20 | `ral_test` | RAL frontdoor/backdoor smoke test |

---

## 3. SVA Plan

| File | Assertion / Cover | Purpose |
|---|---|---|
| `UVMTB/spi/spi_cs_sck_sva.sv` | `CS_SCK_CHK`, `CS_SCK_COV` | CS falling edge to first SCK edge delay |
| `UVMTB/spi/spi_sck_speed_sva.sv` | `SCK_SPEED_CHK`, `SCK_SPEED_COV` | SCK period for each divider |
| `UVMTB/spi/spi_sck_cs_sva.sv` | `SCK_CS_CHK` | Last SCK edge to CS release delay |
| `UVMTB/spi/spi_reset_sva.sv` | `SPI_RESET_CHK` | SPI pins idle during reset |
| `UVMTB/axi/axi_reset_sva.sv` | `AXI_RESET_CHK` | AXI slave outputs idle during reset |
| `UVMTB/axi/axi_reset_sva.sv` | `AXI_RESET_RELEASE_CHK` | AXI slave remains idle after reset release |

---

## 4. RAL Plan

| Register group | Access | Verification method |
|---|---|---|
| `start`, config registers, `mosi_data` | WO | RAL frontdoor write + backdoor peek |
| `busy` | RO | RAL frontdoor read in idle state |
| `miso_data` | RO | RAL backdoor poke/read path smoke |

Implementation:

- Register model: `UVMTB/ral/axi_spi_reg_block.sv`
- Adapter: `UVMTB/ral/axi_spi_reg_adapter.sv`
- Test: `UVMTB/test/ral_test.sv`
- Map connection: existing AXI sequencer through `reg_model.map.set_sequencer(...)`

---

## 5. Coverage Summary

| Coverage | Target | Result / Note |
|---|---:|---|
| Functional coverage | 100% | `cg_spi_frame` cross bins hit |
| SVA / cover property | 100% | Project assertions and cover properties covered |
| DUT line coverage | ~90% | Remaining items documented as waiver |
| DUT branch coverage | ~90% | Remaining items documented as waiver |

Waiver examples:

- Write-only configuration registers are not frontdoor-readable by design.
- Invalid/default address branches are low-value for this project and may stay waived.
- UVM library internal assertions without attempts are not counted as project SVA misses.

---

## 6. Remaining / Optional Work

| Priority | Item | Rationale |
|---:|---|---|
| P1 | AXI read/write concurrency | Optional protocol stress; requires driver architecture change for true concurrent read/write transactions |
| P2 | MISO model expansion | Current directed MISO tests are sufficient for project demo; future work can expand across all mode/word_len combinations |
| P2 | Replace fixed delays with event waits | Improve test robustness, e.g. wait on CS/busy instead of hard-coded `#` delays |

Current project conclusion: for an internship-oriented verification project, the key items are covered. The remaining work is useful polish, not a blocker for project completeness.
