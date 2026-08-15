diff --git a/drivers/can/can_syscalls.c b/drivers/can/can_syscalls.c
index 1234567..abcdef0 100644
--- a/drivers/can/can_syscalls.c
+++ b/drivers/can/can_syscalls.c
@@ -1,6 +1,7 @@
 /*
  * Copyright (c) 2018 Alexander Wachter
  * SPDX-License-Identifier: Apache-2.0
+ * Modified by Giorgio — syscall safety improvements and refactoring
  */

 #include <zephyr/internal/syscall_handler.h>
@@ -10,6 +11,42 @@
 #include <zephyr/drivers/can.h>

+/*
+ * Common helper for verifying CAN timing calculations.
+ * This reduces duplicated logic between classic CAN and CAN-FD timing syscalls.
+ */
+static inline int verify_can_timing_common(
+    const struct device *dev,
+    struct can_timing *res,
+    uint32_t bitrate,
+    uint16_t sample_pnt,
+    bool fd_mode)
+{
+    struct can_timing tmp;
+    int err;
+
+    /* Ensure the driver supports timing operations */
+    K_OOPS(K_SYSCALL_DRIVER_CAN(dev, get_core_clock));
+
+    /* Validate user-mode read access */
+    K_OOPS(K_SYSCALL_MEMORY_READ(res, sizeof(*res)));
+
+    /* Copy timing structure from user-mode to kernel-mode */
+    K_OOPS(k_usermode_from_copy(&tmp, res, sizeof(tmp)));
+
+    /* Dispatch to the correct implementation */
+    err = fd_mode ?
+        z_impl_can_calc_timing_data(dev, &tmp, bitrate, sample_pnt) :
+        z_impl_can_calc_timing(dev, &tmp, bitrate, sample_pnt);
+
+    /* Copy results back to user-mode */
+    K_OOPS(k_usermode_to_copy(res, &tmp, sizeof(*res)));
+
+    return err;
+}
+
 static int z_vrfy_can_calc_timing(const struct device *dev, struct can_timing *res,
                                   uint32_t bitrate, uint16_t sample_pnt)
 {
@@ -17,11 +54,8 @@ static int z_vrfy_can_calc_timing(const struct device *dev, struct can_timing *r
     struct can_timing res_copy;
     int err;

-    K_OOPS(K_SYSCALL_DRIVER_CAN(dev, get_core_clock));
-    K_OOPS(k_usermode_from_copy(&res_copy, res, sizeof(res_copy)));
-
-    err = z_impl_can_calc_timing(dev, &res_copy, bitrate, sample_pnt);
-    K_OOPS(k_usermode_to_copy(res, &res_copy, sizeof(*res)));
+    /* Use new common helper */
+    err = verify_can_timing_common(dev, res, bitrate, sample_pnt, false);

     return err;
 }
@@ -120,7 +154,7 @@ static inline int z_vrfy_can_set_bitrate_data(const struct device *dev,
                                               uint32_t bitrate_data)
 {
-    K_OOPS(K_SYSCALL_DRIVER_CAN(dev, set_timing_data));
+    /* Correct capability check: match the actual implementation */
+    K_OOPS(K_SYSCALL_DRIVER_CAN(dev, set_bitrate_data));

     return z_impl_can_set_bitrate_data(dev, bitrate_data);
 }
@@ -210,7 +244,12 @@ static inline int z_vrfy_can_send(const struct device *dev,
     K_OOPS(K_SYSCALL_DRIVER_CAN(dev, send));
     K_OOPS(k_usermode_from_copy(&frame_copy, frame, sizeof(frame_copy)));
-    K_OOPS(K_SYSCALL_VERIFY_MSG(callback == NULL, "callbacks may not be set from user mode"));
+
+    /* Improved error message for clarity */
+    K_OOPS(K_SYSCALL_VERIFY_MSG(
+        callback == NULL,
+        "User-mode cannot register CAN TX callbacks; use kernel-mode APIs instead"
+    ));

     return z_impl_can_send(dev, &frame_copy, timeout, callback, user_data);
 }
