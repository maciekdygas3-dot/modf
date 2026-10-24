
plugins {
    id 'fabric-loom' version '1.10-SNAPSHOT'
}

version = project.mod_version
group = project.maven_group

base {
    archivesName = project.archives_base_name
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    mappings "net.fabricmc:yarn:${project.yarn_mappings}:v2"
    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
    modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
}

processResources {
    inputs.property "version", project.version
    filesMatching("fabric.mod.json") {
        expand "version": inputs.properties.version
    }
}

tasks.withType(JavaCompile).configureEach {
    it.options.release = 21
}

java {
    withSourcesJar()
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

package com.example.targethud;

import net.fabricmc.api.ClientModInitializer;
import net.fabricmc.fabric.api.client.event.lifecycle.v1.ClientTickEvents;
import net.fabricmc.fabric.api.client.keybinding.v1.KeyBindingHelper;
import net.fabricmc.fabric.api.client.rendering.v1.HudRenderCallback;
import net.fabricmc.fabric.api.event.player.AttackEntityCallback;
import net.minecraft.client.MinecraftClient;
import net.minecraft.client.option.KeyBinding;
import net.minecraft.client.util.InputUtil;
import net.minecraft.entity.LivingEntity;
import net.minecraft.util.ActionResult;
import org.lwjgl.glfw.GLFW;

public class TargetHudMod implements ClientModInitializer {

    private static KeyBinding openGui;

    @Override
    public void onInitializeClient() {
        openGui = KeyBindingHelper.registerKeyBinding(new KeyBinding(
                "key.targethud.gui", InputUtil.Type.KEYSYM,
                GLFW.GLFW_KEY_RIGHT_SHIFT, "category.targethud"));

        ClientTickEvents.END_CLIENT_TICK.register(mc -> {
            while (openGui.wasPressed()) mc.setScreen(new ClickGuiScreen());
        });

        ClientTickEvents.END_CLIENT_TICK.register(Visuals::tick);

        AttackEntityCallback.EVENT.register((player, world, hand, entity, hit) -> {
            if (world.isClient && entity instanceof LivingEntity
                    && player == MinecraftClient.getInstance().player) {
                Visuals.onHit(entity);
            }
            return ActionResult.PASS;
        });

        HudRenderCallback.EVENT.register(Visuals::render);
    }
}

package com.example.targethud;

public class Module {
    public final String name;
    public final Category category;
    public boolean enabled;
    public float anim; // animacja przelacznika 0..1

    public Module(String name, Category category, boolean enabled) {
        this.name = name;
        this.category = category;
        this.enabled = enabled;
        this.anim = enabled ? 1f : 0f;
    }

    public void toggle() {
        enabled = !enabled;
    }
}

package com.example.targethud;

public enum Category {
    COMBAT("Combat"), HUD("HUD"), ANIMATIONS("Animations"),
    SOUND("Sound"), WORLD("World"), CLIENT("Client");

    public final String label;

    Category(String label) {
        this.label = label;
    }
}

package com.example.targethud;

import net.minecraft.client.MinecraftClient;
import net.minecraft.client.gui.DrawContext;
import net.minecraft.client.gui.screen.ingame.InventoryScreen;
import net.minecraft.client.render.RenderTickCounter;
import net.minecraft.client.world.ClientWorld;
import net.minecraft.entity.Entity;
import net.minecraft.entity.LivingEntity;
import net.minecraft.particle.ParticleTypes;
import net.minecraft.sound.SoundCategory;
import net.minecraft.sound.SoundEvents;
import net.minecraft.util.math.MathHelper;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

public class Visuals {

    // ---- stan Target HUD ----
    private static LivingEntity target;
    private static long targetSeen;
    private static float shownHealth;

    // ---- stan trafien / combo ----
    private static long hitTime;
    private static long lastHit;
    private static int combo;

    // ---- stan Kill Effect / Sahur Kill ----
    private static LivingEntity lastVictim;
    private static long victimTime;
    private static final List<KillFx> effects = new ArrayList<>();
    private static final List<SahurFx> sahurFx = new ArrayList<>();

    /** Wywolywane z AttackEntityCallback, gdy uderzasz byt. */
    public static void onHit(Entity e) {
        long now = System.currentTimeMillis();
        hitTime = now;
        combo = (now - lastHit > 3000) ? 1 : combo + 1;
        lastHit = now;

        if (e instanceof LivingEntity le) {
            lastVictim = le;
            victimTime = now;
        }

        MinecraftClient mc = MinecraftClient.getInstance();
        if (ModuleManager.HIT_PARTICLES.enabled && mc.world != null) {
            ThreadLocalRandom r = ThreadLocalRandom.current();
            for (int i = 0; i < 14; i++) {
                mc.world.addParticle(ParticleTypes.CRIT,
                        e.getX(), e.getBodyY(0.5), e.getZ(),
                        (r.nextDouble() - 0.5) * 0.6, r.nextDouble() * 0.4,
                        (r.nextDouble() - 0.5) * 0.6);
            }
        }
    }

    /** Wywolywane co tick klienta (ClientTickEvents.END_CLIENT_TICK). */
    public static void tick(MinecraftClient mc) {
        if (mc.world == null || mc.player == null) {
            effects.clear();
            sahurFx.forEach(SahurFx::cleanup);
            sahurFx.clear();
            lastVictim = null;
            return;
        }

        boolean anyKillFx = ModuleManager.KILL_EFFECT.enabled || ModuleManager.SAHUR_KILL.enabled;
        if (anyKillFx && lastVictim != null) {
            if (System.currentTimeMillis() - victimTime > 5000) {
                lastVictim = null; // za dawno, to nie Twoje zabojstwo
            } else if (!lastVictim.isAlive()) {
                if (mc.player.distanceTo(lastVictim) < 12) {
                    if (ModuleManager.KILL_EFFECT.enabled) {
                        effects.add(new KillFx(lastVictim.getX(), lastVictim.getY(),
                                lastVictim.getZ(), lastVictim.getHeight()));
                    }
                    if (ModuleManager.SAHUR_KILL.enabled) {
                        sahurFx.add(new SahurFx(mc, lastVictim));
                    }
                }
                lastVictim = null;
            }
        }

        effects.removeIf(fx -> fx.tick(mc.world));
        sahurFx.removeIf(SahurFx::tick);
    }

    /** Wywolywane z HudRenderCallback. */
    public static void render(DrawContext ctx, RenderTickCounter tick) {
        MinecraftClient mc = MinecraftClient.getInstance();
        if (mc.player == null || mc.options.hudHidden) return;

        if (ModuleManager.WATERMARK.enabled) watermark(ctx, mc);
        if (ModuleManager.HIT_MARKER.enabled) hitMarker(ctx);
        if (ModuleManager.COMBO_COUNTER.enabled) comboCounter(ctx, mc);
        if (ModuleManager.TARGET_HUD.enabled) targetHud(ctx, mc);
    }

    private static void watermark(DrawContext ctx, MinecraftClient mc) {
        ctx.drawText(mc.textRenderer, "aura", 6, 6, 0xFF4C8DFF, true);
        ctx.drawText(mc.textRenderer, mc.getCurrentFps() + " FPS", 6, 17, 0xFFAABBDD, true);
    }

    private static void hitMarker(DrawContext ctx) {
        long dt = System.currentTimeMillis() - hitTime;
        if (dt > 300) return;
        int alpha = (int) (255 * (1f - dt / 300f));
        int col = (alpha << 24) | 0xFFFFFF;
        int cx = ctx.getScaledWindowWidth() / 2, cy = ctx.getScaledWindowHeight() / 2;
        for (int i = 3; i <= 7; i++) {
            ctx.fill(cx + i,     cy + i,     cx + i + 1, cy + i + 1, col);
            ctx.fill(cx - i - 1, cy + i,     cx - i,     cy + i + 1, col);
            ctx.fill(cx + i,     cy - i - 1, cx + i + 1, cy - i,     col);
            ctx.fill(cx - i - 1, cy - i - 1, cx - i,     cy - i,     col);
        }
    }

    private static void comboCounter(DrawContext ctx, MinecraftClient mc) {
        if (combo == 0 || System.currentTimeMillis() - lastHit > 3000) {
            combo = 0;
            return;
        }
        String s = "Combo: " + combo;
        int x = ctx.getScaledWindowWidth() / 2 - mc.textRenderer.getWidth(s) / 2;
        int y = ctx.getScaledWindowHeight() / 2 + 22;
        ctx.drawText(mc.textRenderer, s, x, y, 0xFFFFFFFF, true);
    }

    private static void targetHud(DrawContext ctx, MinecraftClient mc) {
        if (mc.targetedEntity instanceof LivingEntity le && le != mc.player) {
            if (le != target) shownHealth = le.getHealth();
            target = le;
            targetSeen = System.currentTimeMillis();
        }
        if (target == null || !target.isAlive()
                || System.currentTimeMillis() - targetSeen > 3000) {
            target = null;
            return;
        }

        int w = 150, h = 52;
        int x = ctx.getScaledWindowWidth() / 2 + 25;
        int y = ctx.getScaledWindowHeight() / 2 + 15;

        ctx.fill(x, y, x + w, y + h, 0xB0101018);
        ctx.drawBorder(x, y, w, h, 0xFF55AAFF);
        InventoryScreen.drawEntity(ctx, x + 3, y + 3, x + 43, y + h - 3,
                20, 0.0625F, x + 23, y + 15, target);
        ctx.drawText(mc.textRenderer, target.getName(), x + 48, y + 7, 0xFFFFFFFF, true);

        float hp = target.getHealth(), max = target.getMaxHealth();
        shownHealth += (hp - shownHealth) * 0.15f;
        float ratio = MathHelper.clamp(shownHealth / max, 0f, 1f);
        int bx = x + 48, by = y + 22, bw = w - 54, bh = 8;
        ctx.fill(bx - 1, by - 1, bx + bw + 1, by + bh + 1, 0xFF000000);
        ctx.fill(bx, by, bx + bw, by + bh, 0xFF333333);
        ctx.fill(bx, by, bx + (int) (bw * ratio), by + bh,
                0xFF000000 | MathHelper.hsvToRgb(ratio / 3f, 1f, 1f));

        ctx.drawText(mc.textRenderer, String.format("%.1f / %.1f HP", hp, max),
                bx, by + 12, 0xFFDDDDDD, true);
        String dist = String.format("%.1fm", mc.player.distanceTo(target));
        ctx.drawText(mc.textRenderer, dist, x + w - 6 - mc.textRenderer.getWidth(dist),
                y + 7, 0xFF88CCFF, true);
    }

    // ---------------------------------------------------------------
    // Prosty Kill Effect: wybuch + spirala czasteczek
    // ---------------------------------------------------------------
    private static class KillFx {
        private final double x, y, z;
        private final float height;
        private int age = 0;

        KillFx(double x, double y, double z, float height) {
            this.x = x; this.y = y; this.z = z; this.height = height;
        }

        /** Zwraca true, gdy efekt sie skonczyl. */
        boolean tick(ClientWorld w) {
            ThreadLocalRandom r = ThreadLocalRandom.current();

            if (age == 0) {
                for (int i = 0; i < 40; i++) {
                    w.addParticle(ParticleTypes.TOTEM_OF_UNDYING,
                            x, y + height / 2, z,
                            (r.nextDouble() - 0.5) * 0.8,
                            r.nextDouble() * 0.6,
                            (r.nextDouble() - 0.5) * 0.8);
                }
                w.playSound(x, y, z, SoundEvents.ENTITY_FIREWORK_ROCKET_TWINKLE,
                        SoundCategory.PLAYERS, 1.0f, 1.0f, false);
            }

            double progress = age / 30.0;
            double yy = y + progress * (height + 0.8);
            for (int s = 0; s < 2; s++) {
                double a = age * 0.5 + s * Math.PI;
                w.addParticle(ParticleTypes.END_ROD,
                        x + Math.cos(a) * 0.8, yy, z + Math.sin(a) * 0.8,
                        0, 0.02, 0);
                w.addParticle(ParticleTypes.SOUL_FIRE_FLAME,
                        x + Math.cos(a) * 0.5, yy - 0.2, z + Math.sin(a) * 0.5,
                        0, 0.01, 0);
            }

            return ++age > 30;
        }
    }
}

package com.example.targethud;

import net.minecraft.client.gui.DrawContext;
import net.minecraft.client.gui.screen.Screen;
import net.minecraft.client.gui.widget.TextFieldWidget;
import net.minecraft.text.Text;

import java.util.ArrayList;
import java.util.List;

public class ClickGuiScreen extends Screen {
    private static final int PW = 130, ROW = 22, HEAD = 26, GAP = 10, TOP = 60;

    private record Hit(int x, int y, int w, int h, Module m) {}

    private final List<Hit> hits = new ArrayList<>();
    private TextFieldWidget search;

    public ClickGuiScreen() {
        super(Text.literal("Click GUI"));
    }

    @Override
    protected void init() {
        search = new TextFieldWidget(textRenderer, width - 190, 16, 160, 18, Text.literal("Search"));
        search.setPlaceholder(Text.literal("Type to filter..."));
        addDrawableChild(search);
    }

    @Override
    public void render(DrawContext ctx, int mx, int my, float delta) {
        super.render(ctx, mx, my, delta); // rozmyte tlo + wyszukiwarka
        hits.clear();

        ctx.drawText(textRenderer, "aura", 24, 20, 0xFFFFFFFF, true);
        ctx.drawText(textRenderer, "VISUALS", 24, 32, 0xFF8899BB, false);

        Category[] cats = Category.values();
        int x = (width - (cats.length * PW + (cats.length - 1) * GAP)) / 2;
        String q = search.getText().toLowerCase();

        for (Category c : cats) {
            List<Module> mods = ModuleManager.byCategory(c).stream()
                    .filter(m -> q.isEmpty() || m.name.toLowerCase().contains(q)).toList();
            int h = HEAD + mods.size() * ROW + 6;

            ctx.fill(x, TOP, x + PW, TOP + h, 0xC0182236);
            ctx.drawBorder(x, TOP, PW, h, 0x30FFFFFF);
            ctx.drawText(textRenderer, c.label, x + 10, TOP + 9, 0xFFFFFFFF, true);
            ctx.fill(x + 8, TOP + HEAD - 4, x + PW - 8, TOP + HEAD - 3, 0x25FFFFFF);

            int y = TOP + HEAD;
            for (Module m : mods) {
                boolean hover = mx >= x && mx < x + PW && my >= y && my < y + ROW;
                if (hover) ctx.fill(x + 4, y, x + PW - 4, y + ROW - 2, 0x28FFFFFF);

                ctx.drawText(textRenderer, m.name, x + 12, y + 7,
                        m.enabled ? 0xFFFFFFFF : 0xFF8F9BB3, true);

                m.anim += ((m.enabled ? 1f : 0f) - m.anim) * 0.25f;
                int sx = x + PW - 36, sy = y + 5;
                ctx.fill(sx, sy, sx + 24, sy + 10, lerp(0xFF3A4458, 0xFF4C8DFF, m.anim));
                int kx = sx + 1 + Math.round(m.anim * 14);
                ctx.fill(kx, sy + 1, kx + 8, sy + 9, 0xFFFFFFFF);

                hits.add(new Hit(x, y, PW, ROW, m));
                y += ROW;
            }
            x += PW + GAP;
        }
    }

    @Override
    public boolean mouseClicked(double mx, double my, int button) {
        if (button == 0) {
            for (Hit h : hits) {
                if (mx >= h.x() && mx < h.x() + h.w() && my >= h.y() && my < h.y() + h.h()) {
                    h.m().toggle();
                    return true;
                }
            }
        }
        return super.mouseClicked(mx, my, button);
    }

    @Override
    public void removed() {
        ModuleManager.save();
    }

    @Override
    public boolean shouldPause() {
        return false;
    }

    private static int lerp(int a, int b, float t) {
        int r = 0;
        for (int s = 24; s >= 0; s -= 8) {
            int ca = (a >> s) & 0xFF, cb = (b >> s) & 0xFF;
            r |= (ca + Math.round((cb - ca) * t)) << s;
        }
        return r;
    }
}

package com.example.targethud;

import com.google.gson.GsonBuilder;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import net.fabricmc.loader.api.FabricLoader;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

public class ModuleManager {
    public static final List<Module> MODULES = new ArrayList<>();

    // dzialajace moduly
    public static final Module TARGET_HUD    = add("Target HUD", Category.HUD, true);
    public static final Module HIT_MARKER    = add("Hit Marker", Category.COMBAT, true);
    public static final Module HIT_PARTICLES = add("Hit Particles", Category.COMBAT, true);
    public static final Module KILL_EFFECT   = add("Kill Effect", Category.COMBAT, false);
    public static final Module SAHUR_KILL    = add("Sahur Kill", Category.COMBAT, true);
    public static final Module COMBO_COUNTER = add("Combo Counter", Category.HUD, true);
    public static final Module WATERMARK     = add("Watermark", Category.HUD, true);

    static {
        // placeholdery, jeszcze bez funkcji
        add("Target ESP", Category.COMBAT, false);
        add("Hit Color", Category.COMBAT, false);
        add("Crosshair", Category.HUD, false);
        add("Hurt Cam", Category.ANIMATIONS, false);
        add("Hit Sounds", Category.SOUND, false);
        add("Jump Circles", Category.WORLD, false);
        add("Fullbright", Category.WORLD, false);
        add("Cosmetics", Category.CLIENT, false);
        load();
    }

    private static Module add(String name, Category cat, boolean on) {
        Module m = new Module(name, cat, on);
        MODULES.add(m);
        return m;
    }

    public static List<Module> byCategory(Category c) {
        return MODULES.stream().filter(m -> m.category == c).toList();
    }

    private static Path file() {
        return FabricLoader.getInstance().getConfigDir().resolve("targethud.json");
    }

    public static void save() {
        try {
            JsonObject o = new JsonObject();
            for (Module m : MODULES) o.addProperty(m.name, m.enabled);
            Files.writeString(file(), new GsonBuilder().setPrettyPrinting().create().toJson(o));
        } catch (Exception ignored) {
        }
    }

    private static void load() {
        try {
            if (!Files.exists(file())) return;
            JsonObject o = JsonParser.parseString(Files.readString(file())).getAsJsonObject();
            for (Module m : MODULES) {
                if (o.has(m.name)) {
                    m.enabled = o.get(m.name).getAsBoolean();
                    m.anim = m.enabled ? 1f : 0f;
                }
            }
        } catch (Exception ignored) {
        }
    }
}

package com.example.targethud;

import com.mojang.authlib.GameProfile;
import net.minecraft.client.MinecraftClient;
import net.minecraft.client.network.OtherClientPlayerEntity;
import net.minecraft.client.world.ClientWorld;
import net.minecraft.component.DataComponentTypes;
import net.minecraft.entity.Entity;
import net.minecraft.entity.EquipmentSlot;
import net.minecraft.entity.LivingEntity;
import net.minecraft.entity.SpawnReason;
import net.minecraft.entity.player.PlayerEntity;
import net.minecraft.entity.player.SkinTextures;
import net.minecraft.item.ItemStack;
import net.minecraft.item.Items;
import net.minecraft.particle.ParticleTypes;
import net.minecraft.sound.SoundCategory;
import net.minecraft.sound.SoundEvent;
import net.minecraft.sound.SoundEvents;
import net.minecraft.util.Hand;
import net.minecraft.util.Identifier;
import net.minecraft.util.math.Vec3d;

import java.util.UUID;
import java.util.concurrent.ThreadLocalRandom;

public class SahurFx {

    private static final Identifier SKIN = Identifier.of("targethud", "textures/entity/sahur.png");
    private static int nextId = -70000; // ujemne ID, zeby nie kolidowaly z prawdziwymi bytami

    /** Falszywy gracz z drewnianym skinem. */
    private static class SahurEntity extends OtherClientPlayerEntity {
        SahurEntity(ClientWorld world) {
            super(world, new GameProfile(UUID.randomUUID(), "Tung_Sahur"));
        }

        @Override
        public SkinTextures getSkinTextures() {
            return new SkinTextures(SKIN, null, null, null, SkinTextures.Model.WIDE, true);
        }
    }

    private final ClientWorld world;
    private final LivingEntity victim;
    private final Entity ghost;       // klon ofiary
    private final SahurEntity sahur;
    private final Vec3d origin;
    private final Vec3d dir;          // kierunek od Sahura do ofiary (poziomo)
    private final float gap;
    private int age = 0;
    private Vec3d vel = Vec3d.ZERO;
    private Vec3d pos;
    private float spin;

    public SahurFx(MinecraftClient mc, LivingEntity victim) {
        this.world = mc.world;
        this.victim = victim;
        this.origin = victim.getPos();
        this.pos = origin;
        this.gap = victim.getWidth() + 1.0f;

        // Sahur staje z boku, patrzac na ofiare
        Vec3d toVictim = origin.subtract(mc.player.getPos());
        Vec3d forward = new Vec3d(toVictim.x, 0, toVictim.z).normalize();
        Vec3d side = new Vec3d(-forward.z, 0, forward.x);
        this.dir = side.multiply(-1);

        // klon ofiary
        Entity g;
        if (victim instanceof PlayerEntity pe) {
            g = new OtherClientPlayerEntity(world, pe.getGameProfile());
        } else {
            g = victim.getType().create(world, SpawnReason.EVENT);
        }
        this.ghost = g;
        if (ghost != null) {
            ghost.setId(nextId--);
            ghost.refreshPositionAndAngles(origin.x, origin.y, origin.z, victim.getYaw(), 0);
            ghost.setNoGravity(true);
            world.addEntity(ghost);
            victim.setInvisible(true); // chowamy prawdziwe cialo
        }

        // Sahur z kijem
        this.sahur = new SahurEntity(world);
        sahur.setId(nextId--);
        sahur.setNoGravity(true);
        ItemStack bat = new ItemStack(Items.STICK);
        bat.set(DataComponentTypes.ITEM_MODEL, Identifier.of("targethud", "sahur_bat"));
        sahur.equipStack(EquipmentSlot.MAINHAND, bat);
        Vec3d start = origin.add(side.multiply(gap + 2.0));
        sahur.refreshPositionAndAngles(start.x, start.y, start.z, yawTo(dir), 0);
        world.addEntity(sahur);
    }

    private static float yawTo(Vec3d d) {
        return (float) Math.toDegrees(Math.atan2(-d.x, d.z));
    }

    private void snd(SoundEvent s, float vol, float pitch, Vec3d at) {
        world.playSound(at.x, at.y, at.z, s, SoundCategory.PLAYERS, vol, pitch, false);
    }

    /** Zwraca true, gdy efekt sie skonczyl. */
    public boolean tick() {
        ThreadLocalRandom r = ThreadLocalRandom.current();
        Vec3d side = dir.multiply(-1);

        // Sahur podchodzi (ticki 0..10)
        double dist = gap + 2.0 - Math.min(age, 10) * (2.0 / 10.0);
        Vec3d sPos = origin.add(side.multiply(dist));
        float yaw = yawTo(dir);
        sahur.refreshPositionAndAngles(sPos.x, sPos.y, sPos.z, yaw, 0);
        sahur.setHeadYaw(yaw);
        sahur.setBodyYaw(yaw);

        // tung, tung, tung
        if (age == 3 || age == 7 || age == 11) {
            snd(SoundEvents.BLOCK_WOOD_HIT, 1.5f, 0.55f, sPos);
            sahur.swingHand(Hand.MAIN_HAND);
        }

        // czwarte uderzenie: bonk i odlot
        if (age == 15) {
            sahur.swingHand(Hand.MAIN_HAND);
            snd(SoundEvents.ENTITY_PLAYER_ATTACK_CRIT, 1.5f, 0.7f, pos);
            snd(SoundEvents.ENTITY_PLAYER_ATTACK_KNOCKBACK, 1.5f, 0.8f, pos);
            vel = dir.multiply(1.1).add(0, 0.85, 0);
            for (int i = 0; i < 25; i++) {
                world.addParticle(ParticleTypes.CRIT,
                        pos.x, pos.y + victim.getHeight(), pos.z,
                        (r.nextDouble() - 0.5), r.nextDouble() * 0.6, (r.nextDouble() - 0.5));
            }
        }

        // lot klonu
        if (age >= 15 && ghost != null) {
            vel = new Vec3d(vel.x * 0.98, vel.y - 0.06, vel.z * 0.98);
            pos = pos.add(vel);
            spin += 40f;
            ghost.refreshPositionAndAngles(pos.x, pos.y, pos.z, spin, 0);
            ghost.setYaw(spin);
            if (ghost instanceof LivingEntity le) {
                le.setBodyYaw(spin);
                le.setHeadYaw(spin);
            }
            world.addParticle(ParticleTypes.CLOUD, pos.x, pos.y + 0.5, pos.z, 0, 0, 0);
            if (age % 3 == 0) {
                world.addParticle(ParticleTypes.CRIT, pos.x, pos.y + 0.5, pos.z, 0, 0, 0);
            }
        }

        // koniec: dym i sprzatanie
        if (age == 45) {
            for (int i = 0; i < 20; i++) {
                world.addParticle(ParticleTypes.POOF, pos.x, pos.y + 0.5, pos.z,
                        (r.nextDouble() - 0.5) * 0.4, r.nextDouble() * 0.3, (r.nextDouble() - 0.5) * 0.4);
                world.addParticle(ParticleTypes.POOF, sPos.x, sPos.y + 1, sPos.z,
                        (r.nextDouble() - 0.5) * 0.3, r.nextDouble() * 0.3, (r.nextDouble() - 0.5) * 0.3);
            }
            snd(SoundEvents.ENTITY_FIREWORK_ROCKET_BLAST, 0.8f, 1.2f, pos);
            cleanup();
            return true;
        }

        age++;
        return false;
    }

    public void cleanup() {
        if (ghost != null) ghost.discard();
        sahur.discard();
        if (!victim.isRemoved()) victim.setInvisible(false);
    }
}

{
  "schemaVersion": 1,
  "id": "targethud",
  "version": "${version}",
  "name": "Target HUD",
  "description": "Target HUD, Click GUI, hit effects and Tung Tung Sahur kill effect.",
  "authors": ["you"],
  "license": "MIT",
  "environment": "client",
  "entrypoints": {
    "client": ["com.example.targethud.TargetHudMod"]
  },
  "depends": {
    "fabricloader": ">=0.16.0",
    "minecraft": "~1.21.4",
    "java": ">=21",
    "fabric-api": "*"
  }
}

{
  "model": {
    "type": "minecraft:model",
    "model": "targethud:item/sahur_bat"
  }
}
{
  "parent": "minecraft:item/handheld",
  "textures": {
    "layer0": "targethud:item/sahur_bat"
  },
  "display": {
    "thirdperson_righthand": {
      "rotation": [
        0,
        -90,
        55
      ],
      "translation": [
        0,
        5,
        1
      ],
      "scale": [
        1.5,
        1.5,
        1.5
      ]
    }
  }
}
{
  "key.targethud.gui": "Open Click GUI",
  "category.targethud": "Target HUD"
}

# Target HUD (Fabric 1.21.4)

Jeden mod: Click GUI (Prawy Shift), Target HUD, Hit Marker, Hit Particles,
Combo Counter, Watermark, Kill Effect i Sahur Kill (Tung Tung Tung Sahur).

## Jak zbudowac plik .jar

Wymagania: Java 21 (JDK) i internet (Gradle pobiera Minecrafta i Fabric API).

### Sposob 1: IntelliJ IDEA (najlatwiej)
1. File > Open > wybierz ten folder (TargetHudMod).
2. Poczekaj, az Gradle zaimportuje projekt (na dole pasek postepu).
3. Po prawej: Gradle > Tasks > build > build (dwuklik).
4. Gotowy mod: build/libs/targethud-1.0.0.jar (NIE ten z "-sources").

### Sposob 2: z konsoli, jesli masz zainstalowany Gradle
    gradle wrapper --gradle-version 8.12.1
    ./gradlew build        (Windows: gradlew.bat build)

### Sposob 3: szablon Fabric
1. Wygeneruj projekt na https://fabricmc.net/develop/template (1.21.4, Yarn, Java 21).
2. Podmien w nim folder src/ folderem src/ z tego archiwum
   oraz plik fabric.mod.json (mod id: targethud).
3. Uruchom: ./gradlew build

## Instalacja
Wrzuc targethud-1.0.0.jar do folderu mods razem z Fabric API (1.21.4).
Mod jest kliencki, wiec dziala tez na serwerach bez moda.

## Sterowanie
Prawy Shift: otwiera Click GUI. Ustawienia zapisuja sie w config/targethud.json.

org.gradle.jvmargs=-Xmx2G
org.gradle.parallel=true

# Fabric (Minecraft 1.21.4)
minecraft_version=1.21.4
yarn_mappings=1.21.4+build.8
loader_version=0.16.10
fabric_version=0.119.4+1.21.4

# Mod
mod_version=1.0.0
maven_group=com.example
archives_base_name=targethud

distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.12.1-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists

pluginManagement {
    repositories {
        maven { name = 'Fabric'; url = 'https://maven.fabricmc.net/' }
        mavenCentral()
        gradlePluginPortal()
    }
}
