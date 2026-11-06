# UHD `num_send_frames` Configuration for NI USRP-2901

## Purpose

This document explains the UHD `num_send_frames` transport setting, why it was increased for the USB-connected NI USRP-2901, and how to use the validated configuration for continuous transmission at 10 MS/s.

## Recommended configuration

Use the following UHD device arguments:

```text
type=b200,num_send_frames=256
```

Keep the required transmit sample rate at:

```text
10 MS/s
```

`num_send_frames=256` does not change the sample rate, carrier frequency, signal bandwidth, or generated waveform.

## Meaning of `num_send_frames=256`

`num_send_frames` specifies the number of asynchronous USB transmit-transfer buffers allocated and managed by UHD between the host computer and the USRP.

The transmit path is:

```text
Application samples
        |
        v
UHD TX streamer
        |
        v
USB send-transfer buffers
        |
        v
Windows USB driver / USB 3 controller
        |
        v
NI USRP-2901 transmit FIFO
        |
        v
DAC and RF output
```

A larger transfer pool allows UHD to keep more USB operations ready or in flight. This helps the stream survive short Windows scheduling delays and temporary USB latency without allowing the USRP transmit FIFO to become empty.

A concise engineering description is:

> `num_send_frames=256` configures UHD's USB transport to allocate 256 simultaneous asynchronous TX transfer buffers. This increases host-to-device transfer concurrency and protects the USRP TX FIFO from brief host or USB scheduling delays.

## Related parameters

The following settings serve different purposes:

| Parameter | Meaning |
|---|---|
| `tx_rate=10000000` | Transmit sample rate of 10 MS/s |
| `TX spp=2040` | Number of samples carried in one UHD TX packet |
| `num_send_frames=256` | Number of USB TX transfer buffers available to UHD |
| `sc16` | Two signed 16-bit components per complex sample; four bytes per complex sample |
| `sc8` | Two signed 8-bit components per complex sample; two bytes per complex sample |

`num_send_frames=256` does **not** mean 256 samples, 256 RF frames, or a 256-sample hardware FIFO.

## Approximate transport capacity

For the observed configuration:

```text
TX spp                 = 2,040 complex samples
sc16 payload           = 4 bytes per complex sample
Approximate frame data = 2,040 x 4 = 8,160 bytes
```

Therefore, 256 frames represent approximately:

```text
256 x 8,160 bytes = 2,088,960 bytes
```

At 10 MS/s using `sc16`, the nominal sample payload is:

```text
10,000,000 x 4 bytes = 40,000,000 bytes/second
```

The frame pool is therefore comparable to roughly 52 ms of sample payload. This is only an explanatory approximation: buffers may be free, being filled, queued, transferring, or completed. It is not a guaranteed 52 ms hardware prefill.

## Validation results

The NI USRP-2901 was detected as a B210-class device operating over USB 3.

An earlier 10 MS/s test using fewer/default send frames reported:

```text
Duration:             20 seconds
Underruns detected:   21
```

With 128 send frames, the result improved to:

```text
Duration:             120 seconds
Underruns detected:   2
Sequence errors:      0
Late commands:        0
TX timeouts:          0
```

With 256 send frames, the required 10 MS/s rate passed a ten-minute test:

```text
Duration:                 600 seconds
Transmitted samples:      6,003,116,160
Underruns detected:       0
TX sequence errors:       0
Late commands:            0
TX timeouts:              0
```

This demonstrates that the USB link had sufficient average bandwidth at 10 MS/s, but the smaller transfer pool did not provide enough protection against brief Windows/USB scheduling latency.

## UHD benchmark command

In PowerShell, define the benchmark executable:

```powershell
$benchmark = "C:\Program Files\PothosSDR\lib\uhd\examples\benchmark_rate.exe"
```

Run the validated configuration for ten minutes:

```powershell
& $benchmark `
  --args "type=b200,num_send_frames=256" `
  --tx_rate 10000000 `
  --channels 0 `
  --duration 600
```

Run a 25-minute qualification test:

```powershell
& $benchmark `
  --args "type=b200,num_send_frames=256" `
  --tx_rate 10000000 `
  --channels 0 `
  --duration 1500
```

Typing only the following in PowerShell does not configure anything:

```powershell
num_send_frames=256
```

The option must be included in the UHD device-argument string supplied when the device is opened.

## C and C++ UHD API configuration

Pass the setting to UHD when creating the USRP:

```cpp
const std::string deviceArgs =
    "type=b200,serial=31420E7,num_send_frames=256";

uhd_usrp_make(&usrp, deviceArgs.c_str());
```

If only one B200-series device is connected, the serial number may be omitted:

```cpp
const std::string deviceArgs =
    "type=b200,num_send_frames=256";
```

The setting is a device/transport initialization argument. It must be supplied before the TX streamer is created.

## NavikEye configuration

Configure the UHD resource string as:

```json
{
  "resource": "type=b200,num_send_frames=256"
}
```

If the exact NI USRP-2901 must be selected:

```json
{
  "resource": "type=b200,serial=31420E7,num_send_frames=256"
}
```

Retain the application sample rate of 10 MS/s.

## NI-USRP LabVIEW API limitation

The standard NI-USRP LabVIEW API does not expose UHD's `num_send_frames` device argument. There is no direct `num_send_frames=256` property in:

- `niUSRP Open Tx Session.vi`
- `niUSRP Configure Signal.vi`
- `niUSRP Write Tx Data.vi`
- the standard `niUSRP Property Node`

For an application using only the NI-USRP LabVIEW API, use the closest application-level approach:

1. Use a producer-consumer architecture.
2. Generate sample blocks in a producer loop.
3. Maintain several ready blocks in a LabVIEW queue.
4. Use a dedicated transmitter loop containing `Dequeue Element` and `niUSRP Write Tx Data`.
5. Keep file access, graph updates, logging, and waveform generation out of the transmitter loop.
6. Write larger blocks and keep `end of data?` set to `FALSE` during continuous transmission.
7. Prefer the I16 host data type where supported to reduce CDB conversion and memory overhead.

For the 10 MS/s NI USRP-2901 stream, reasonable write sizes based on a 2,040-sample packet multiple are:

| Samples per write | Approximate signal duration |
|---:|---:|
| 102,000 | 10.2 ms |
| 204,000 | 20.4 ms |
| 510,000 | 51.0 ms |

Start with 204,000 samples per write and maintain five to ten prepared blocks in the LabVIEW queue.

If the exact UHD behavior validated by the benchmark is mandatory, use UHD directly, such as through the NavikEye C++ application or a custom UHD-based DLL called from LabVIEW. Do not open the same USRP simultaneously through NI-USRP and UHD.

## Prefill and continuous buffering

`num_send_frames` is not equivalent to the explicitly allocated onboard waveform buffer used by NI-RFSG.

For UHD streaming, use two complementary mechanisms:

1. A timed first TX packet schedules transmission slightly in the future, allowing initial samples to be queued before RF generation starts.
2. An application-side producer-consumer queue keeps generated samples ready for the dedicated UHD sender.

The application should monitor:

- current and minimum software queue depth;
- producer wait time;
- sender wait time;
- duration of each UHD send call;
- requested and returned sample counts;
- partial sends;
- underflow timestamps.

If the software queue reaches zero before an underflow, the generator is starving the sender. If the queue remains populated but an underflow occurs, the cause is in the UHD, USB, driver, or host-scheduling path.

## Final recommendation

For the validated NI USRP-2901 configuration:

```text
Sample rate:       10 MS/s
Device arguments:  type=b200,num_send_frames=256
USB connection:    Direct USB 3 connection
Qualification:     Zero underruns during a 600-second benchmark
```

Keep `num_send_frames=256`. There is currently no evidence that increasing it to 512 is necessary.

