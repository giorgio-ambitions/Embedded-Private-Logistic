diff --git a/can_common.c b/can_common.c
index 1234567..abcdef0 100644
--- a/can_common.c
+++ b/can_common.c
@@ -1,6 +1,7 @@
 /* Copyright (c) 2019 Alexander Wachter
  * SPDX-License-Identifier: Apache-2.0
  */
+/* Patch by Giorgio — CAN timing & safety fixes */

@@ -67,7 +68,12 @@ int z_impl_can_send(const struct device *dev, const struct can_frame *frame,
        if (callback == NULL) {
-               struct can_tx_default_cb_ctx ctx;
+               /* Ensure callback context is safe even if driver is async */
+               static struct can_tx_default_cb_ctx ctx;
+               /* NOTE: static context avoids use-after-scope; still not ideal
+                * for concurrent sends, but prevents UB. A proper fix requires
+                * API guarantees or dynamic allocation.
+                */

                k_sem_init(&ctx.done, 0, 1);

@@ -158,12 +164,12 @@ static int update_sample_pnt(uint32_t total_tq, uint32_t sample_pnt, struct can_timing *res,
-    const struct can_timing *main, const struct can_timing *max)
+    const struct can_timing *min, const struct can_timing *max)
 {
-    uint16_t tseg1_max = max->phase_seg1 + max->prop_seg;
-    uint16_t tseg1_min = min->phase_seg1 + min->prop_seg;
+    uint16_t tseg1_max = max->phase_seg1 + max->prop_seg;
+    uint16_t tseg1_min = min->phase_seg1 + min->prop_seg;

@@ -176,7 +182,7 @@ static int update_sample_pnt(uint32_t total_tq, uint32_t sample_pnt, struct can_timing *res,
-    tseg2 = total_tq - (total_tq * sample_pnt) / 1000;
+    tseg2 = total_tq - DIV_ROUND_CLOSEST(total_tq * sample_pnt, 1000);

@@ -214,7 +220,7 @@ static int update_sample_pnt(uint32_t total_tq, uint32_t sample_pnt, struct can_timing *res,
-    res->sjw = MIN(res->phase_seg1, res->phase_seg2 / 2);
+    res->sjw = MIN(res->phase_seg1, res->phase_seg2);

@@ -260,7 +266,7 @@ static int can_calc_timing_internal(const struct device *dev, struct can_timing *res,
-    for (int prescaler = MAX(core_clock / (total_tq * bitrate), min->prescaler);
+    for (uint32_t prescaler = MAX(core_clock / (total_tq * bitrate), min->prescaler);

@@ -268,6 +274,11 @@ static int can_calc_timing_internal(const struct device *dev, struct can_timing *res,
-        if (core_clock % (prescaler * bitrate)) {
+        /* Prevent overflow and invalid divisors */
+        if (prescaler == 0 || bitrate == 0 ||
+            prescaler > max->prescaler ||
+            (uint64_t)prescaler * bitrate > core_clock ||
+            core_clock % (prescaler * bitrate)) {

@@ -355,6 +366,10 @@ int z_impl_can_set_timing(const struct device *dev,
-    err = check_timing_in_range(timing, min, max);
+    /* Extended validation: ensure full bit-time consistency */
+    err = check_timing_in_range(timing, min, max);
+    if (timing->prop_seg + timing->phase_seg1 + timing->phase_seg2 + CAN_SYNC_SEG > max->prop_seg + max->phase_seg1 + max->phase_seg2 + CAN_SYNC_SEG) {
+        return -ENOTSUP;
+    }

