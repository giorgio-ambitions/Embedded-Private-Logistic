GitHub Review – Zephyr CAN Loopback Driver (can_loopback.c)
Overall summary The loopback driver is generally well‑structured, consistent with Zephyr’s CAN API, and easy to follow. Concurrency is handled correctly, error paths are clear, and the implementation matches expected loopback semantics. Below are detailed notes and improvement suggestions.
✔️ Architecture & Concurrency
    • The separation between TX thread, message queue, and filter processing is clean and follows Zephyr driver patterns.
    • Mutex usage around filter iteration is correct and prevents race conditions between TX thread and filter management.
    • The TX thread design (blocking on k_msgq_get with K_FOREVER) is appropriate for a loopback device.
Suggestion: Consider documenting the callback order (TX callback before loopback RX callbacks). This is correct per Zephyr semantics, but making it explicit helps avoid user confusion.
✔️ Send Path (can_loopback_send)
    • Proper validation of CAN and CAN‑FD flags.
    • Correct handling of DLC limits (CAN_MAX_DLC / CANFD_MAX_DLC).
    • Returns meaningful error codes (-ENOTSUP, -EINVAL, -ENETDOWN, -EAGAIN).
    • Message queue insertion is straightforward and safe.
Suggestion: The temporary copy of the frame into loopback_frame.frame is fine, but consider clarifying immutability expectations in comments.
✔️ Receive Path & Filters
    • Filter matching uses can_frame_matches_filter, which is correct and consistent with other drivers.
    • Filter allocation via get_free_filter() is simple and effective.
    • Mutex protection around filter updates is correct.
Suggestion: receive_frame() copies the frame into frame_tmp before invoking the callback. If immutability is not required, this copy could be avoided. If immutability is required, consider documenting why.
✔️ Driver State Management
    • start() / stop() correctly manage the started flag.
    • stop() purges the TX queue, which is appropriate for a loopback device.
    • get_state() returns consistent CAN states (ERROR_ACTIVE vs STOPPED).
Suggestion: can_loopback_set_state_change_callback() is currently a stub. It may be worth either implementing minimal notifications or documenting that loopback devices do not support state change events.
✔️ Initialization
    • Thread creation via k_thread_create is correct and uses the configured stack size.
    • Message queue initialization is correct.
    • Filter array is properly reset.
Suggestion: If thread creation fails, the driver returns -1. Consider returning a more specific Zephyr error code for consistency.
✔️ Device Tree & API Registration
    • DT_DRV_COMPAT zephyr_can_loopback is correct.
    • CAN_DEVICE_DT_INST_DEFINE is used properly.
    • Capability reporting (get_capabilities) is accurate.
⭐ Final Assessment
This driver is clean, correct, and consistent with Zephyr’s CAN subsystem. The implementation is easy to follow, and concurrency is handled safely. The suggestions above are minor improvements rather than required fixes.
Great work overall.
