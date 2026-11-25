I've now checked the audit against your v1_28 file, and its core findings hold up. It overstates how severe a few are, and some of what it calls bugs are your own design choices.

Valid (confirmed in v1_28)

• Rejection row mismatch: Real. ScoreCore only gives +REJ at a location (rejLoc: OB tap, OTZ tap, or within InpGradeNearATR of the slow EMA). EntryCheckUpdate (line 4278) sets f.rej from RecentRejection alone. So a pin bar in no-man's land shows PASS on the panel while scoring 0. The ENTRY DECISION row still follows the real stage, so it's a misleading row, not a false signal. The audit's "High" is too strong, but it defeats the purpose of a panel meant to mirror the live checks. The fix is outside the CORE block, so RunSelfTest() is unaffected. In EntryCheckUpdate, after f.zoneTap is set, add:
double extATR = MathAbs(closeC - slowC) / atr;
bool rejLoc = !InpRejNeedsLocation || f.zoneTap || extATR <= InpGradeNearATR;
f.rej = atrOk && RecentRejection(f.isBuy, atr) && rejLoc;
Here closeC and atr must be read first, and rejLoc and extATR need to be computed after atrOk is known.
• Mitigated order blocks evicting live ones (line 3231): Still unfixed. It still evicts g_ob[0] regardless of state.
• OTZ anchor: Still has no minimum leg size, so it can lock onto tiny swings in a range.
• Deposit drift in daily limits: Still there. Add only DEAL_TYPE_BALANCE (not CREDIT).
• Grade math: This one is more relevant than in v1_25, because v1_28 defaults InpMinGrade = 1. Compact trend gives at most 1 point and B needs 2. So one soft penalty (-SAR, -RSI_WEAK, -M15) kills an otherwise good setup unless two structural points fire.
• Preview vs live H4Counter: Accurate. g_ctActive is only refreshed at arm time and at the quality-gate step, while UpdatePreview recomputes it every bar. I didn't check every place that reads g_ctActive, so I'd treat it as display-level.

Valid, but it's your design

• Three-bar pipeline, -M15 during an M15 reversal, fractal lag: All deliberate and documented in comments. They're settings to tune, not defects.
• Expiry lock: By design, but it interacts badly with the grade default, which the audit missed. A setup that is armed but blocked on GRADE keeps aging, expires after InpConfirmMaxAgeBars (8), and then stays locked until a close through the fast EMA. With InpMinGrade = 1, a borderline setup can therefore burn its 8 bars and lock you out of the whole move. That's the most important thing to know if you want more trades.

Not an issue

• The sweep window edge case and H4 age (display-only).

The two worth fixing first are the OB eviction and the rejection row, about 15 minutes of work together. If you want more entries, the grade and lock interaction is the bigger lever. Want me to patch v1_28 with those fixes and tests for the new behavior?# vortex-apex-m15-reversal
VORTEX APEX with M15 Trend Reversal Detection - Complete MQ5 EA
