
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

package com.example.targethud.mixin;

import com.example.targethud.ModuleManager;
import net.minecraft.client.gui.hud.InGameHud;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfo;

@Mixin(InGameHud.class)
public class InGameHudMixin {

    /** Chowa wanilijny celownik, gdy wlaczony jest nasz Crosshair. */
    @Inject(method = "renderCrosshair", at = @At("HEAD"), cancellable = true, require = 0)
    private void targethud$hideVanillaCrosshair(CallbackInfo ci) {
        if (ModuleManager.CROSSHAIR.enabled) ci.cancel();
    }
}

package com.example.targethud.mixin;

import com.example.targethud.ModuleManager;
import net.minecraft.client.render.GameRenderer;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfo;

@Mixin(GameRenderer.class)
public class GameRendererMixin {

    /** Wylacza przechylanie kamery po otrzymaniu obrazen. */
    @Inject(method = "tiltViewWhenHurt", at = @At("HEAD"), cancellable = true, require = 0)
    private void targethud$noHurtCam(CallbackInfo ci) {
        if (ModuleManager.NO_HURT_CAM.enabled) ci.cancel();
    }
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
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class TargetHudMod implements ClientModInitializer {

    public static final Logger LOGGER = LoggerFactory.getLogger("targethud");

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
import net.minecraft.entity.decoration.ArmorStandEntity;
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

    // ---- stan Jump Circles / Target ESP ----
    private static boolean renderErrorLogged;
    private static boolean wasOnGround;
    private static final List<RingFx> rings = new ArrayList<>();
    private static LivingEntity espTarget;
    private static long espSeen;
    private static int espTicks;

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

        if (ModuleManager.HIT_SOUNDS.enabled && mc.world != null && mc.player != null) {
            mc.world.playSound(mc.player.getX(), mc.player.getY(), mc.player.getZ(),
                    SoundEvents.ENTITY_ARROW_HIT_PLAYER, SoundCategory.PLAYERS, 0.8f, 1.4f, false);
        }

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
                        try {
                            sahurFx.add(new SahurFx(mc, lastVictim));
                        } catch (Throwable t) {
                            TargetHudMod.LOGGER.warn("Sahur Kill: nie udalo sie utworzyc efektu", t);
                            lastVictim.setInvisible(false);
                        }
                    }
                }
                lastVictim = null;
            }
        }

        if (mc.player.hurtTime > 0) combo = 0; // combo konczy sie po otrzymaniu obrazen

        // Jump Circles: pierscien czasteczek w momencie skoku
        boolean onGround = mc.player.isOnGround();
        if (ModuleManager.JUMP_CIRCLES.enabled && wasOnGround && !onGround
                && mc.player.getVelocity().y > 0.1) {
            rings.add(new RingFx(mc.player.getX(), mc.player.getY(), mc.player.getZ()));
        }
        wasOnGround = onGround;
        try {
            rings.removeIf(r -> r.tick(mc.world));
        } catch (Throwable t) {
            rings.clear();
            TargetHudMod.LOGGER.warn("Jump Circles: blad efektu", t);
        }

        // Target ESP: obracajace sie czasteczki wokol celu
        if (mc.targetedEntity instanceof LivingEntity le && le != mc.player
                && !(le instanceof ArmorStandEntity)) {
            espTarget = le;
            espSeen = System.currentTimeMillis();
        }
        if (ModuleManager.TARGET_ESP.enabled && espTarget != null) {
            if (!espTarget.isAlive() || System.currentTimeMillis() - espSeen > 3000) {
                espTarget = null;
            } else {
                espTicks++;
                double a = espTicks * 0.35;
                double rad = espTarget.getWidth() * 0.9 + 0.2;
                double yy = espTarget.getY() + (Math.sin(espTicks * 0.12) + 1) / 2 * espTarget.getHeight();
                for (int s = 0; s < 2; s++) {
                    double ang = a + s * Math.PI;
                    mc.world.addParticle(ParticleTypes.END_ROD,
                            espTarget.getX() + Math.cos(ang) * rad, yy,
                            espTarget.getZ() + Math.sin(ang) * rad, 0, 0, 0);
                }
            }
        }

        try {
            effects.removeIf(fx -> fx.tick(mc.world));
        } catch (Throwable t) {
            effects.clear();
            TargetHudMod.LOGGER.warn("Kill Effect: blad efektu", t);
        }
        try {
            sahurFx.removeIf(SahurFx::tick);
        } catch (Throwable t) {
            for (SahurFx f : sahurFx) {
                try { f.cleanup(); } catch (Throwable ignored) { }
            }
            sahurFx.clear();
            TargetHudMod.LOGGER.warn("Sahur Kill: blad efektu, wylaczono biezace efekty", t);
        }
    }

    /** Wywolywane z HudRenderCallback. */
    public static void render(DrawContext ctx, RenderTickCounter tick) {
        MinecraftClient mc = MinecraftClient.getInstance();
        if (mc.player == null || mc.options.hudHidden) return;

        try {
            if (ModuleManager.WATERMARK.enabled) watermark(ctx, mc);
            if (ModuleManager.CROSSHAIR.enabled) crosshair(ctx, mc);
            if (ModuleManager.HIT_MARKER.enabled) hitMarker(ctx);
            if (ModuleManager.COMBO_COUNTER.enabled) comboCounter(ctx, mc);
            if (ModuleManager.TARGET_HUD.enabled) targetHud(ctx, mc);
        } catch (Throwable t) {
            if (!renderErrorLogged) {
                renderErrorLogged = true;
                TargetHudMod.LOGGER.error("Blad renderowania HUD", t);
            }
        }
    }

    private static void crosshair(DrawContext ctx, MinecraftClient mc) {
        int cx = ctx.getScaledWindowWidth() / 2, cy = ctx.getScaledWindowHeight() / 2;
        int gap = 3, len = 5;
        int col = mc.targetedEntity instanceof LivingEntity ? 0xFFFF4040 : 0xFFFFFFFF;
        int sh = 0x90000000;

        // cien
        ctx.fill(cx - gap - len - 1, cy - 1, cx - gap + 1, cy + 2, sh);
        ctx.fill(cx + gap, cy - 1, cx + gap + len + 1, cy + 2, sh);
        ctx.fill(cx - 1, cy - gap - len - 1, cx + 2, cy - gap + 1, sh);
        ctx.fill(cx - 1, cy + gap, cx + 2, cy + gap + len + 1, sh);
        // kreski
        ctx.fill(cx - gap - len, cy, cx - gap, cy + 1, col);
        ctx.fill(cx + gap + 1, cy, cx + gap + len + 1, cy + 1, col);
        ctx.fill(cx, cy - gap - len, cx + 1, cy - gap, col);
        ctx.fill(cx, cy + gap + 1, cx + 1, cy + gap + len + 1, col);
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
        if (mc.targetedEntity instanceof LivingEntity le && le != mc.player
                && !(le instanceof ArmorStandEntity)) {
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
    /** Rozszerzajacy sie pierscien czasteczek (Jump Circles). */
    private static class RingFx {
        private final double x, y, z;
        private int age = 0;

        RingFx(double x, double y, double z) {
            this.x = x; this.y = y; this.z = z;
        }

        boolean tick(ClientWorld w) {
            double radius = 0.4 + age * 0.14;
            int n = 28;
            for (int i = 0; i < n; i++) {
                double a = 2 * Math.PI * i / n;
                w.addParticle(ParticleTypes.END_ROD,
                        x + Math.cos(a) * radius, y + 0.05, z + Math.sin(a) * radius, 0, 0, 0);
            }
            return ++age > 8;
        }
    }

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
import org.lwjgl.glfw.GLFW;

import java.util.ArrayList;
import java.util.List;

public class ClickGuiScreen extends Screen {
    private static final int PW = 130, ROW = 22, HEAD = 26, GAP = 10, TOP = 60;
    private static final int ACCENT = 0xFF4C8DFF;

    // pozycje paneli zapamietane na czas sesji (mozna je przeciagac)
    private static final int[] PX = new int[Category.values().length];
    private static final int[] PY = new int[Category.values().length];
    private static boolean placed = false;

    private record Hit(int x, int y, int w, int h, Module m) {}
    private record Head(int x, int y, int w, int h, int index) {}

    private final List<Hit> hits = new ArrayList<>();
    private final List<Head> heads = new ArrayList<>();
    private TextFieldWidget search;
    private long openedAt;
    private int dragging = -1, dragOffX, dragOffY;

    public ClickGuiScreen() {
        super(Text.literal("Click GUI"));
    }

    @Override
    protected void init() {
        openedAt = System.currentTimeMillis();
        search = new TextFieldWidget(textRenderer, width - 190, 16, 160, 18, Text.literal("Search"));
        search.setPlaceholder(Text.literal("Type to filter..."));
        addDrawableChild(search);

        if (!placed) {
            int n = Category.values().length;
            int x = (width - (n * PW + (n - 1) * GAP)) / 2;
            for (int i = 0; i < n; i++) {
                PX[i] = x + i * (PW + GAP);
                PY[i] = TOP;
            }
            placed = true;
        }
    }

    @Override
    public void render(DrawContext ctx, int mx, int my, float delta) {
        super.render(ctx, mx, my, delta); // rozmyte tlo + wyszukiwarka
        hits.clear();
        heads.clear();

        // animacja otwarcia (wjazd + pojawienie sie)
        float t = Math.min(1f, (System.currentTimeMillis() - openedAt) / 180f);
        float ease = 1f - (1f - t) * (1f - t);
        int slide = Math.round((1f - ease) * 14);

        ctx.drawText(textRenderer, "aura", 24, 20, a(0xFFFFFFFF, ease), true);
        ctx.drawText(textRenderer, "VISUALS", 24, 32, a(0xFF8899BB, ease), false);

        Category[] cats = Category.values();
        String q = search.getText().toLowerCase();

        for (int i = 0; i < cats.length; i++) {
            Category c = cats[i];
            List<Module> mods = ModuleManager.byCategory(c).stream()
                    .filter(m -> q.isEmpty() || m.name.toLowerCase().contains(q)).toList();
            int h = HEAD + mods.size() * ROW + 6;

            int x = Math.max(0, Math.min(PX[i], width - PW));
            int y = Math.max(0, Math.min(PY[i], height - HEAD)) + slide;

            roundRect(ctx, x, y, PW, h, a(0xD0161F30, ease));
            ctx.fill(x + 3, y, x + PW - 3, y + 1, a(ACCENT, ease)); // akcent u gory
            ctx.drawText(textRenderer, c.label, x + 10, y + 9, a(0xFFFFFFFF, ease), true);
            String cnt = String.valueOf(mods.size());
            ctx.drawText(textRenderer, cnt, x + PW - 10 - textRenderer.getWidth(cnt), y + 9,
                    a(0xFF6F7C96, ease), false);
            ctx.fill(x + 8, y + HEAD - 4, x + PW - 8, y + HEAD - 3, a(0x25FFFFFF, ease));
            heads.add(new Head(x, y, PW, HEAD, i));

            int ry = y + HEAD;
            for (Module m : mods) {
                boolean hover = dragging < 0 && mx >= x && mx < x + PW && my >= ry && my < ry + ROW;
                if (hover) roundRect(ctx, x + 4, ry, PW - 8, ROW - 2, a(0x28FFFFFF, ease));
                if (m.enabled) ctx.fill(x + 4, ry + 3, x + 6, ry + ROW - 5, a(ACCENT, ease));

                ctx.drawText(textRenderer, m.name, x + 12, ry + 7,
                        a(m.enabled ? 0xFFFFFFFF : 0xFF8F9BB3, ease), true);

                // animowany przelacznik
                m.anim += ((m.enabled ? 1f : 0f) - m.anim) * 0.25f;
                int sx = x + PW - 36, sy = ry + 5;
                roundRect(ctx, sx, sy, 24, 10, a(lerp(0xFF3A4458, ACCENT, m.anim), ease));
                int kx = sx + 2 + Math.round(m.anim * 12);
                roundRect(ctx, kx, sy + 1, 8, 8, a(0xFFFFFFFF, ease));

                hits.add(new Hit(x, ry, PW, ROW, m));
                ry += ROW;
            }
        }

        ctx.drawCenteredTextWithShadow(textRenderer,
                "Right Shift / ESC - zamknij   |   przeciagnij naglowek, zeby przesunac panel",
                width / 2, height - 16, a(0xFF8899BB, ease));
    }

    @Override
    public boolean mouseClicked(double mx, double my, int button) {
        if (button == 0) {
            for (Head hd : heads) {
                if (mx >= hd.x() && mx < hd.x() + hd.w() && my >= hd.y() && my < hd.y() + hd.h()) {
                    dragging = hd.index();
                    PX[dragging] = hd.x();
                    PY[dragging] = hd.y();
                    dragOffX = (int) mx - hd.x();
                    dragOffY = (int) my - hd.y();
                    return true;
                }
            }
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
    public boolean mouseDragged(double mx, double my, int button, double dx, double dy) {
        if (dragging >= 0 && button == 0) {
            PX[dragging] = (int) mx - dragOffX;
            PY[dragging] = (int) my - dragOffY;
            return true;
        }
        return super.mouseDragged(mx, my, button, dx, dy);
    }

    @Override
    public boolean mouseReleased(double mx, double my, int button) {
        dragging = -1;
        return super.mouseReleased(mx, my, button);
    }

    @Override
    public boolean keyPressed(int keyCode, int scanCode, int modifiers) {
        if (keyCode == GLFW.GLFW_KEY_RIGHT_SHIFT && !search.isFocused()) {
            close();
            return true;
        }
        return super.keyPressed(keyCode, scanCode, modifiers);
    }

    @Override
    public void removed() {
        ModuleManager.save();
    }

    @Override
    public boolean shouldPause() {
        return false;
    }

    /** Prostokat z lekko zaokraglonymi rogami (bez nakladajacych sie wypelnien). */
    private static void roundRect(DrawContext c, int x, int y, int w, int h, int color) {
        c.fill(x + 2, y, x + w - 2, y + h, color);
        c.fill(x + 1, y + 1, x + 2, y + h - 1, color);
        c.fill(x + w - 2, y + 1, x + w - 1, y + h - 1, color);
        c.fill(x, y + 2, x + 1, y + h - 2, color);
        c.fill(x + w - 1, y + 2, x + w, y + h - 2, color);
    }

    /** Mnozy przezroczystosc koloru przez f (0..1). */
    private static int a(int color, float f) {
        int al = Math.round(((color >>> 24) & 0xFF) * f);
        return (al << 24) | (color & 0xFFFFFF);
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

    // ---- Combat ----
    public static final Module HIT_MARKER    = add("Hit Marker", Category.COMBAT, true);
    public static final Module HIT_PARTICLES = add("Hit Particles", Category.COMBAT, true);
    public static final Module TARGET_ESP    = add("Target ESP", Category.COMBAT, true);
    public static final Module KILL_EFFECT   = add("Kill Effect", Category.COMBAT, false);
    public static final Module SAHUR_KILL    = add("Sahur Kill", Category.COMBAT, true);

    // ---- HUD ----
    public static final Module TARGET_HUD    = add("Target HUD", Category.HUD, true);
    public static final Module WATERMARK     = add("Watermark", Category.HUD, true);
    public static final Module COMBO_COUNTER = add("Combo Counter", Category.HUD, true);
    public static final Module CROSSHAIR     = add("Crosshair", Category.HUD, false);

    // ---- Animations ----
    public static final Module NO_HURT_CAM   = add("No Hurt Cam", Category.ANIMATIONS, false);

    // ---- Sound ----
    public static final Module HIT_SOUNDS    = add("Hit Sounds", Category.SOUND, true);

    // ---- World ----
    public static final Module JUMP_CIRCLES  = add("Jump Circles", Category.WORLD, true);

    static {
        // placeholdery, jeszcze bez funkcji
        add("Hit Color", Category.COMBAT, false);
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
  "required": true,
  "minVersion": "0.8",
  "package": "com.example.targethud.mixin",
  "compatibilityLevel": "JAVA_21",
  "client": [
    "InGameHudMixin",
    "GameRendererMixin"
  ],
  "injectors": {
    "defaultRequire": 0
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
  "mixins": ["targethud.mixins.json"],
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

name: build
on:
  push:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.12.1
      - run: gradle build
      - uses: actions/upload-artifact@v4
        with:
          name: targethud-mod
          path: build/libs/*.jar

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

## Moduly (Prawy Shift)
Dzialaja: Target HUD, Hit Marker, Hit Particles, Combo Counter, Watermark,
Kill Effect, Sahur Kill, Hit Sounds, Crosshair, No Hurt Cam, Jump Circles, Target ESP.
Placeholdery: Hit Color, Fullbright, Cosmetics.

## Click GUI
Panele mozna przeciagac za naglowek. Right Shift lub ESC zamyka menu.
Efekty maja zabezpieczenia: jesli jakis efekt zglosi blad, zostanie wylaczony
(blad trafi do logs/latest.log), zamiast zawieszac gre.

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
