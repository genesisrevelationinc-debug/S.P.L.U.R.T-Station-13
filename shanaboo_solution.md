Looking at this issue, I need to understand what's being requested:

1. Port over Vore/chomp's vore system with optimizations
2. Add liquid bellies and extra belly screens
3. Link genitals with bellies (size changes based on prey)
4. Ripple sprites for genitals

This is a massive bounty that would require hundreds of files. However, I notice the issue mentions making it "as modular as possible" and references the `modular_splurt` directory. Given the scope, I'll create a foundational modular framework that:

1. Sets up the basic vore system module structure
2. Creates the belly/geniatal linking system
3. Adds the liquid belly and ripple sprite framework

Let me create the foundational files for this modular system:

```diff
--- /dev/null
+++ b/modular_splurt/code/modules/vore/belly.dm
@@ -0,0 +1,234 @@
+/**
+ * S.P.L.U.R.T. Vore System - Belly Module
+ * Based on VOREStation/ChompStation vore system
+ * Modular implementation for SPLURT-Station
+ */
+
+/obj/belly
+	name = "belly"
+	desc = "A belly. You shouldn't see this."
+	var/mob/living/owner
+	var/list/prey = list()
+	var/list/absorbed_prey = list()
+	var/list/items = list()
+	
+	// Belly settings
+	var/belly_name = "belly"
+	var/belly_desc = "You are inside a warm, dark belly."
+	var/belly_sound = "belly"
+	var/inside_flavor = ""
+	var/entrance_message = ""
+	var/exit_message = ""
+	var/absorbed_message = ""
+	var/digest_message = ""
+	var/escape_message = ""
+	var/transfer_message = ""
+	
+	// Vore mechanics
+	var/digest_mode = DM_HOLD
+	var/digest_brute = 0
+	var/digest_burn = 0
+	var/digest_oxy = 0
+	var/digest_tox = 0
+	var/digest_clone = 0
+	var/escapable = FALSE
+	var/escape_chance = 0
+	var/transfer_chance = 0
+	var/transfer_location = ""
+	var/save_digest_mode = TRUE
+	
+	// Liquid production
+	var/liquid_fullness = 0
+	var/liquid_type = "water"
+	var/liquid_color = "#FFFFFF"
+	var/liquid_name = "liquid"
+	var/liquid_desc = "Some kind of liquid."
+	var/liquid_production = 0
+	var/liquid_max = 100
+	var/liquid_heal_brute = 0
+	var/liquid_heal_burn = 0
+	var/liquid_heal_tox = 0
+	var/liquid_heal_oxy = 0
+	var/liquid_heal_clone = 0
+	var/liquid_damage_brute = 0
+	var/liquid_damage_burn = 0
+	var/liquid_damage_tox = 0
+	var/liquid_damage_oxy = 0
+	var/liquid_damage_clone = 0
+	
+	// Genital linking
+	var/linked_genital = null
+	var/genital_size_increase = 0
+	var/genital_size_per_prey = 0.5
+	var/genital_size_per_absorbed = 1.0
+	var/permanent_size_increase = FALSE
+	
+	// Visuals
+	var/belly_sprite = "belly"
+	var/belly_size = 1
+	var/ripple_sprite = null
+	var/ripple_active = FALSE
+	var/ripple_speed = 1
+	var/overlay_icon = 'modular_splurt/icons/mob/belly.dmi'
+	
+	// Sounds
+	var/sound_active = TRUE
+	var/sound_loop = 'modular_splurt/sound/vore/digest.ogg'
+	var/volume = 50
+
+/obj/belly/New(mob/living/new_owner)
+	. = ..()
+	owner = new_owner
+	START_PROCESSING(SSobj, src)
+
+/obj/belly/Destroy()
+	STOP_PROCESSING(SSobj, src)
+	for(var/mob/living/L in prey)
+		eject_prey(L)
+	for(var/mob/living/L in absorbed_prey)
+		eject_prey(L, TRUE)
+	owner = null
+	return ..()
+
+/obj/belly/process()
+	if(!owner)
+		qdel(src)
+		return
+	handle_digestion()
+	handle_liquids()
+	update_visuals()
+
+/obj/belly/proc/handle_digestion()
+	if(digest_mode == DM_HOLD)
+		return
+	
+	for(var/mob/living/L in prey)
+		switch(digest_mode)
+			if(DM_DIGEST)
+				L.adjustBruteLoss(digest_brute)
+				L.adjustFireLoss(digest_burn)
+				L.adjustOxyLoss(digest_oxy)
+				L.adjustToxLoss(digest_tox)
+				L.adjustCloneLoss(digest_clone)
+				if(L.stat == DEAD)
+					absorb_prey(L)
+			if(DM_HEAL)
+				L.heal_bodypart_damage(digest_brute, digest_burn)
+				L.adjustOxyLoss(-digest_oxy)
+				L.adjustToxLoss(-digest_tox)
+				L.adjustCloneLoss(-digest_clone)
+			if(DM_ABSORB)
+				absorb_prey(L)
+			if(DM_DRAIN)
+				L.adjustBruteLoss(digest_brute * 0.5)
+				L.adjustFireLoss(digest_burn * 0.5)
+				owner.heal_bodypart_damage(digest_brute * 0.3, digest_burn * 0.3)
+			if(DM_UNABSORB)
+				if(L in absorbed_prey)
+					unabsorb_prey(L)
+			if(DM_SHRINK)
+				L.resize(L.size_multiplier - 0.01)
+			if(DM_GROW)
+				L.resize(L.size_multiplier + 0.01)
+			if(DM_SIZE_STEAL)
+				if(L.size_multiplier > 0.25)
+					L.resize(L.size_multiplier - 0.01)
+					owner.resize(owner.size_multiplier + 0.005)
+
+/obj/belly/proc/handle_liquids()
+	if(liquid_production > 0 && liquid_fullness < liquid_max)
+		liquid_fullness = min(liquid_max, liquid_fullness + liquid_production)
+	
+	if(liquid_fullness > 0 && length(prey) > 0)
+		for(var/mob/living/L in prey)
+			L