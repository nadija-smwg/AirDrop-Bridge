# UniversalDrop — Risk Register

**Date:** September 2026

---

## Risk Severity Scale

```
Severity = Probability × Impact

CRITICAL: Project-blocking if not mitigated
HIGH:     Significant scope/timeline impact
MEDIUM:   Manageable with engineering effort
LOW:      Minor inconvenience
```

---

## Risk Register

| ID | Risk | Category | Probability | Impact | Severity | Mitigation | Fallback | Status |
|----|------|----------|:-----------:|:------:|:--------:|------------|----------|--------|
| R01 | AWDL requires special Wi-Fi hardware not available on most consumer PCs/phones | Hardware | **Certain** | High | **CRITICAL** | Document compatible hardware; recommend specific USB adapters; architecture supports non-AWDL path | Native UniversalDrop protocol for devices without AWDL hardware | ✅ CONFIRMED |
| R02 | Windows cannot run AWDL natively (no monitor mode, no packet injection) | Platform | **Very High** | High | **CRITICAL** | Linux VM bridge with USB Wi-Fi passthrough; monitor DMA/Wi-Fi Aware | Windows supports native protocol only; AirDrop via bridge | ✅ CONFIRMED |
| R03 | Android AWDL not accessible to third-party apps (Google's impl is private) | Platform | **Very High** | High | **CRITICAL** | Monitor Wi-Fi Aware (DMA) developments; relay bridge via Linux | Native protocol primary; AirDrop via bridge or future standard | ✅ CONFIRMED |
| R04 | Apple changes AirDrop protocol in future iOS updates | Protocol | **Medium** | High | **HIGH** | Version detection in protocol layer; modular architecture for rapid adaptation; maintain compatibility test matrix | Pin to known-working iOS versions; document breakage promptly | 🟡 PROBABLE |
| R05 | Apple-signed identity certificates required for "Contacts Only" mode | Auth | **Certain** | Medium | **HIGH** | Clearly document "Everyone mode only" limitation; guide users to set "Everyone for 10 minutes" | "Everyone" mode is fully functional for file transfer | ✅ CONFIRMED |
| R06 | Large file transfers stall on half-duplex monitor-mode radio | Performance | **Medium** | Medium | **HIGH** | Implement retry/resume; optimize chunk size; test multiple adapters; implement flow control | Accept lower throughput for AWDL path; recommend wired alternative for very large files | 🟡 PROBABLE |
| R07 | iOS rejects BLE advertisements from non-Apple devices | Protocol | **Low** | Very High | **HIGH** | Test on multiple iOS versions; analyze BLE validation logic; fall back to AWDL-only discovery (no BLE trigger) | Receiver starts AWDL without BLE; user must open AirDrop on iPhone first | ❓ UNKNOWN |
| R08 | GPLv3 code accidentally included in codebase | Legal | **Low** | High | **MEDIUM** | Code review policy; `cargo deny` in CI; license scanning; clean-room implementation process | Remove and reimplement any contaminated code | Preventable |
| R09 | AWDL timing precision insufficient on commodity USB Wi-Fi hardware | Hardware | **Medium** | Medium | **MEDIUM** | Test multiple adapters; tune timing parameters empirically; implement drift compensation | Limit to known-good hardware; reduce AW precision requirements | 🧪 EXPERIMENTAL |
| R10 | Cross-platform Rust+JNI/FFI bridge introduces instability or crashes | Engineering | **Medium** | Medium | **MEDIUM** | Extensive integration testing; memory safety at boundary; error handling in JNI | Use higher-level IPC (e.g., Unix socket/TCP) between Rust and platform code | 🟡 PROBABLE |
| R11 | Android vendor restricts custom BLE manufacturer data in advertisements | Platform | **Medium** | Medium | **MEDIUM** | Test on multiple vendors (Samsung, Google, OnePlus, Xiaomi); use standard BLE APIs | Limit AirDrop BLE to tested devices; native protocol doesn't need Apple BLE | 🟡 PROBABLE |
| R12 | Security vulnerability discovered in protocol implementation | Security | **Medium** | Very High | **HIGH** | Fuzz testing; security audit; use well-audited crypto libraries (ring, rustls); minimize attack surface | Responsible disclosure; rapid patch cycle | Ongoing |
| R13 | Maintenance complexity across 3+ platforms exceeds available resources | Engineering | **High** | Medium | **MEDIUM** | Shared Rust core minimizes per-platform code; CI testing on all platforms; prioritize Linux and Android | Reduce to 2 platforms if resources limited; defer Windows | Ongoing |
| R14 | EU DMA interoperability (Wi-Fi Aware for AirDrop) delayed or insufficient | Regulatory | **Medium** | Medium | **MEDIUM** | Don't depend solely on DMA; maintain reverse-engineering path; monitor regulatory developments | Continue with AWDL reverse-engineering approach | 🟡 PROBABLE |
| R15 | Apple actively blocks non-Apple AirDrop receivers | Legal/Protocol | **Low** | Very High | **HIGH** | Protocol version detection; rapid adaptation; maintain multiple discovery strategies | Pivot to native protocol; label AirDrop as "experimental" | ❓ UNKNOWN |
| R16 | Performance insufficient compared to native AirDrop | Performance | **Medium** | Low | **MEDIUM** | Benchmark early; optimize hot paths; document expected performance | Accept lower throughput; focus on reliability over speed | 🟡 PROBABLE |
| R17 | CPIO archive format changes between iOS versions | Protocol | **Low** | Medium | **LOW** | Test across iOS versions; implement robust parser with fallbacks | Support multiple CPIO variants; error recovery | 🟡 PROBABLE |
| R18 | iPhone "Everyone for 10 min" timeout too short for setup | UX | **Medium** | Low | **LOW** | Optimize discovery speed; provide clear user instructions; auto-reconnect if dropped | Instruct user to re-enable; consider "try again" flow in UI | ✅ CONFIRMED |
| R19 | Competing project (e.g., Google Quick Share) makes project redundant | Market | **Medium** | Medium | **MEDIUM** | Focus on open-source, privacy-first differentiation; support platforms Quick Share doesn't | Pivot to complementary features; contribute to open standards | 🟡 PROBABLE |
| R20 | Patent litigation risk from Apple | Legal | **Low** | Very High | **MEDIUM** | Focus on interoperability (legally protected in many jurisdictions); consult legal; EU DMA provides protection | Limit distribution to EU jurisdictions; seek legal clarity | ❓ UNKNOWN |
