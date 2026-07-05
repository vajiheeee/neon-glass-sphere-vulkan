// gsnShaderOptions: precompiler="GL_EXT_ray_tracing, recursion: 5, app: 1"
// IMPORTANT: do not remove the line above

/**** ABOUT ****/
// Procedural neon hexagon scene with a ray-traced glass sphere.
// The wall and floor are generated procedurally; the sphere comes from Edit Scene.

/**** COMMON START ****/

struct RayPayloadType {
  vec3 directLight;
  vec3 nextRayOrigin;
  vec3 nextRayDirection;
  vec3 nextFactor;
  int level;
  int hit;
};

struct HitAttributeType {
  vec3 normal;
  vec2 bary;
  vec2 texCoord;
};

RayPayloadType payload;
const float PI = 3.14159265359;

vec3 fresnelSchlick(float cosTheta, vec3 F0) {
  return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}

// Approximate sphere position used for floor shadow and reflection.
const vec3 SPHERE_CENTER = vec3(0.0, -0.42, -1.92);

// ------------------------------------------------------------
// Utility
// ------------------------------------------------------------

float hash21(vec2 p) {
  vec3 p3 = fract(vec3(p.xyx) * 0.1031);
  p3 += dot(p3, p3.yzx + 33.33);
  return fract((p3.x + p3.y) * p3.z);
}

vec2 hash22(vec2 p) {
  float n = sin(dot(p, vec2(41.0, 289.0)));
  return fract(vec2(262144.0, 32768.0) * n);
}

float sdSegment(vec2 p, vec2 a, vec2 b) {
  vec2 pa = p - a;
  vec2 ba = b - a;
  float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
  return length(pa - ba * h);
}

float sdHexagon(vec2 p, float r) {
  const vec3 k = vec3(-0.8660254, 0.5, 0.5773503);
  p = abs(p);
  p -= 2.0 * min(dot(k.xy, p), 0.0) * k.xy;
  p -= vec2(clamp(p.x, -k.z * r, k.z * r), r);
  return length(p) * sign(p.y);
}

// ------------------------------------------------------------
// Stars / dust
// ------------------------------------------------------------

vec3 starField(vec2 uv) {
  vec2 gv = uv * 120.0;
  vec2 id = floor(gv);
  vec2 f = fract(gv) - 0.5;

  float rnd = hash21(id);
  vec2 ofs = hash22(id) - 0.5;

  float d = length(f - ofs * 0.7);

  float smallStar = smoothstep(0.045, 0.0, d);
  smallStar *= step(0.982, rnd);

  float brightStar = smoothstep(0.035, 0.0, d);
  brightStar *= step(0.995, rnd);

  float twinkle = 0.75 + 0.25 * sin(20.0 * rnd);

  vec3 col = vec3(0.0);
  col += vec3(1.0) * smallStar * twinkle * 0.75;
  col += vec3(1.0) * brightStar * 1.40;

  return col;
}

// ------------------------------------------------------------
// Hollow neon hexagons
// ------------------------------------------------------------

vec3 drawHex(vec2 p, vec2 center, float r, vec3 color, float glowStrength) {
  vec2 q = p - center;

  float dOuter = abs(sdHexagon(q, r));
  float outerLine = 1.0 - smoothstep(0.004, 0.010, dOuter);
  float outerGlow = exp(-46.0 * dOuter) * glowStrength;

  float innerR = r - 0.035;
  float dInner = abs(sdHexagon(q, innerR));
  float innerLine = 1.0 - smoothstep(0.004, 0.010, dInner);
  float innerGlow = exp(-46.0 * dInner) * glowStrength * 0.85;

  float insideFill = 1.0 - smoothstep(-0.010, 0.018, sdHexagon(q, innerR - 0.010));

  float centerFade = 1.0 - smoothstep(0.0, innerR, length(q));
  centerFade = mix(0.45, 1.0, centerFade);

  vec3 softColor = mix(color, vec3(1.0), 0.10);

  vec3 col = vec3(0.0);

  vec3 fillColor = mix(vec3(0.0, 0.04, 0.08), color, 0.85);
  col += fillColor * insideFill * centerFade * 0.52;

  col += softColor * outerLine * 0.95;
  col += softColor * innerLine * 0.70;

  col += softColor * outerGlow * 0.95;
  col += softColor * innerGlow * 0.25;

  return col;
}

// ------------------------------------------------------------
// Dark background hexagons
// ------------------------------------------------------------

vec3 drawDarkHex(vec2 p, vec2 center, float r, vec3 edgeColor, vec3 fillColor, float edgeGlow) {
  vec2 q = p - center;

  float d = abs(sdHexagon(q, r));
  float edge = 1.0 - smoothstep(0.005, 0.015, d);
  float fill = 1.0 - smoothstep(0.0, 0.01, sdHexagon(q, r - 0.01));
  float glow = exp(-35.0 * d) * edgeGlow;

  vec3 col = vec3(0.0);
  col += fillColor * fill * 0.35;
  col += edgeColor * edge * 0.55;
  col += edgeColor * glow * 0.12;

  return col;
}

// ------------------------------------------------------------
// Connection lines between hexagons
// ------------------------------------------------------------

vec3 connectHex(vec2 p, vec2 a, vec2 b, vec3 color) {
  float d = sdSegment(p, a, b);

  float line = 1.0 - smoothstep(0.008, 0.018, d);
  float glow = exp(-35.0 * d);

  vec3 col = vec3(0.0);
  col += color * line * 1.05;
  col += color * glow * 0.35;

  return col;
}

// ------------------------------------------------------------
// Circuit paths
// ------------------------------------------------------------

vec3 drawCircuitPath(vec2 p, vec2 a, vec2 b, vec2 c, vec2 d, vec3 color) {
  float s1 = 1.0 - smoothstep(0.004, 0.010, sdSegment(p, a, b));
  float s2 = 1.0 - smoothstep(0.004, 0.010, sdSegment(p, b, c));
  float s3 = 1.0 - smoothstep(0.004, 0.010, sdSegment(p, c, d));

  float g1 = exp(-55.0 * sdSegment(p, a, b));
  float g2 = exp(-55.0 * sdSegment(p, b, c));
  float g3 = exp(-55.0 * sdSegment(p, c, d));

  vec3 col = vec3(0.0);
  col += color * (s1 + s2 + s3) * 0.95;
  col += color * (g1 + g2 + g3) * 0.10;

  float distEnd = length(p - d);
  float ring = 1.0 - smoothstep(0.004, 0.010, abs(distEnd - 0.018));
  float dotGlow = exp(-140.0 * distEnd * distEnd);

  col += color * ring * 1.0;
  col += color * dotGlow * 0.25;

  return col;
}

vec3 drawCircuit(vec2 p) {
  vec3 gold = vec3(1.00, 0.86, 0.10);

  vec3 col = vec3(0.0);

  // Left circuit group.
  col += drawCircuitPath(
    p,
    vec2(-1.42,  0.50),
    vec2(-1.12,  0.50),
    vec2(-1.00,  0.30),
    vec2(-1.00,  0.64),
    gold
  );

  col += drawCircuitPath(
    p,
    vec2(-1.22,  0.45),
    vec2(-1.02,  0.45),
    vec2(-0.92,  0.28),
    vec2(-0.92,  0.56),
    gold
  );

  col += drawCircuitPath(
    p,
    vec2(-1.18,  0.40),
    vec2(-0.98,  0.40),
    vec2(-0.86,  0.56),
    vec2(-0.86,  0.35),
    gold
  );

  // Right circuit group.
  col += drawCircuitPath(
    p,
    vec2(0.10,  0.00),
    vec2(0.40,  0.00),
    vec2(0.50,  0.18),
    vec2(0.50,  0.36),
    gold
  );

  col += drawCircuitPath(
    p,
    vec2(0.22,  0.00),
    vec2(0.48,  0.00),
    vec2(0.58, -0.12),
    vec2(0.58, -0.28),
    gold
  );

  col += drawCircuitPath(
    p,
    vec2(0.34,  0.00),
    vec2(0.66,  0.00),
    vec2(0.80,  0.08),
    vec2(0.98,  0.08),
    gold
  );

  col += drawCircuitPath(
    p,
    vec2(0.26,  0.00),
    vec2(0.14,  0.00),
    vec2(0.44,  0.10),
    vec2(0.44,  0.24),
    gold
  );

  return col;
}

/**** COMMON END ****/

// ------------------------------------------------------------
// Procedural neon wall pattern
// ------------------------------------------------------------

vec3 wallPattern(vec2 p) {
  vec3 col = vec3(0.0);

  p *= 0.55;

  vec2 uvFake = p * 0.25 + 0.5;
  col += starField(uvFake) * 0.8;

  vec3 blue  = vec3(0.0, 0.278, 0.545);
  vec3 cyan  = vec3(0.35, 0.92, 1.00);
  vec3 green = vec3(0.396, 0.886, 0.549);

  // Dark background hexagons.
  col += drawDarkHex(p, vec2(-0.82, -0.06), 0.23, vec3(0.14,0.28,0.58), vec3(0.03,0.06,0.12), 0.22);
  col += drawDarkHex(p, vec2( 0.26,  0.14), 0.18, vec3(0.12,0.25,0.50), vec3(0.02,0.05,0.10), 0.18);
  col += drawDarkHex(p, vec2( 1.38,  0.32), 0.19, vec3(0.10,0.22,0.45), vec3(0.02,0.04,0.09), 0.15);
  col += drawDarkHex(p, vec2( 0.92, -0.18), 0.15, vec3(0.10,0.20,0.42), vec3(0.02,0.04,0.08), 0.12);

  // Side continuation hexagons.
  col += drawHex(p, vec2(-1.45,  0.08), 0.17, blue,  0.70);
  col += drawHex(p, vec2(-1.92,  0.18), 0.15, cyan,  0.55);
  col += drawHex(p, vec2(-1.58, -0.08), 0.15, green,  0.52);

  col += drawHex(p, vec2( 1.15,  0.00), 0.14, blue,  0.72);
  col += drawHex(p, vec2( 1.68, -0.10), 0.10, green,  0.58);
  col += drawHex(p, vec2( 1.88, -0.56), 0.15, cyan,  0.45);

  // Main hexagon groups.
  col += drawHex(p, vec2(-1.05,  0.14), 0.18, cyan,  0.82);
  col += drawHex(p, vec2(-0.73, -0.04), 0.18, blue,  0.95);
  col += drawHex(p, vec2(-0.41,  0.14), 0.18, green,  0.90);
  col += drawHex(p, vec2( 0.24,  0.12), 0.18, cyan, 0.95);
  col += drawHex(p, vec2( 0.56, -0.06), 0.18, green, 0.80);
  col += drawHex(p, vec2( 0.88,  0.12), 0.18, blue,  0.92);

  // Small upper hexagons.
  col += drawHex(p, vec2(-1.36,  0.36), 0.12, cyan,  0.48);
  col += drawHex(p, vec2(-0.06,  0.38), 0.13, blue,  0.45);
  col += drawHex(p, vec2( 1.18,  0.38), 0.13, blue,  0.44);
  col += drawHex(p, vec2( 1.58,  0.18), 0.12, blue,  0.36);

  // Connections.
  col += connectHex(p, vec2(-1.05,  0.14), vec2(-0.73, -0.04), cyan);
  col += connectHex(p, vec2(-0.73, -0.04), vec2(-0.41,  0.14), blue);
  col += connectHex(p, vec2(-1.05,  0.14), vec2(-0.41,  0.14), blue * 0.75);

  col += connectHex(p, vec2( 0.24,  0.12), vec2( 0.56, -0.06), green);
  col += connectHex(p, vec2( 0.56, -0.06), vec2( 0.88,  0.12), blue);
  col += connectHex(p, vec2( 0.24,  0.12), vec2( 0.88,  0.12), green * 0.75);

  col += connectHex(p, vec2(-0.41, 0.14), vec2(0.24, 0.12), blue * 0.45);

  // Extra small hexagons for depth.
  col += drawHex(p, vec2(-0.95,  0.62), 0.075, cyan,  0.45);
  col += drawHex(p, vec2(-0.35,  0.58), 0.070, blue,  0.38);
  col += drawHex(p, vec2( 0.28,  0.64), 0.075, green, 0.42);
  col += drawHex(p, vec2( 0.82,  0.56), 0.070, blue,  0.36);

  col += drawHex(p, vec2(-0.78, -0.56), 0.075, blue,  0.35);
  col += drawHex(p, vec2(-0.12, -0.62), 0.065, cyan,  0.30);
  col += drawHex(p, vec2( 0.48, -0.58), 0.070, green, 0.36);
  col += drawHex(p, vec2( 1.02, -0.50), 0.065, blue,  0.32);
  col += drawHex(p, vec2(-1.98, 0.12), 0.16, cyan, 0.45);
  col += drawHex(p, vec2( 2.02, 0.18), 0.16, blue, 0.45);

  // Faint background hexagons.
  col += drawDarkHex(p, vec2(-1.20,  0.32), 0.18, vec3(0.10,0.20,0.45), vec3(0.02,0.03,0.07), 0.10);
  col += drawDarkHex(p, vec2( 1.12,  0.36), 0.18, vec3(0.08,0.18,0.40), vec3(0.02,0.03,0.06), 0.10);

  // Extra lower small hexagons.
  col += drawHex(p, vec2(-0.92, -0.48), 0.075, blue,  0.28);
  col += drawHex(p, vec2(-0.30, -0.52), 0.070, cyan,  0.24);
  col += drawHex(p, vec2( 0.28, -0.50), 0.075, green, 0.24);
  col += drawHex(p, vec2( 0.88, -0.46), 0.070, blue,  0.24);

  // Extra upper small hexagons.
  col += drawHex(p, vec2(-0.62,  0.62), 0.075, blue,  0.38);
  col += drawHex(p, vec2(-0.12,  0.68), 0.085, cyan,  0.36);
  col += drawHex(p, vec2( 0.42,  0.66), 0.080, green, 0.34);
  col += drawHex(p, vec2( 0.96,  0.62), 0.075, blue,  0.32);

  col += drawHex(p, vec2(-0.30,  0.52), 0.11, blue,  0.28);
  col += drawHex(p, vec2( 0.68,  0.50), 0.11, green, 0.26);

  // Golden circuit layer.
  col += drawCircuit(p);

  return col;
}

// ------------------------------------------------------------
// Main
// ------------------------------------------------------------

/**** RAY GENERATION SHADER ****/
void main() { /**** RAY GENERATION SHADER ****/

  vec2 uv = (vec2(gl_LaunchIDEXT.xy) + 0.5) / vec2(gl_LaunchSizeEXT.xy);
  float aspect = float(gl_LaunchSizeEXT.x) / float(gl_LaunchSizeEXT.y);

  // Screen coordinates.
  vec2 screen = vec2((uv.x - 0.5) * 2.0 * aspect, (uv.y - 0.5) * 2.0);

  // Camera.
  vec3 ro = vec3(0.0, 0.0, 0.0);
  ///vec3 rd = normalize(vec3(screen.x, screen.y, -1.55));
  vec3 rd = normalize(vec3(screen.x, screen.y - 0.16, -1.55)); // slight downward view

  vec3 col = vec3(0.0);

  float nearestT = 1e20;
  vec3 hitPos = vec3(0.0);
  int surfaceID = -1;

  // Room settings.
  float radius = 3.2;
  float floorY = -1.45;
  float topY   =  2.2;

  // --------------------------------------------------
  // Curved wall
  // --------------------------------------------------

  float A = rd.x * rd.x + rd.z * rd.z;
  float B = 2.0 * (ro.x * rd.x + ro.z * rd.z);
  float C = ro.x * ro.x + ro.z * ro.z - radius * radius;

  float disc = B * B - 4.0 * A * C;

  if (disc > 0.0) {
    float t = (-B + sqrt(disc)) / (2.0 * A);
    vec3 h = ro + rd * t;

    if (t > 0.0 && h.z < 0.2 && h.y > floorY && h.y < topY) {
      nearestT = t;
      hitPos = h;
      surfaceID = 0;
    }
  }

  // --------------------------------------------------
  // Floor
  // --------------------------------------------------

  if (rd.y < -0.001) {
    float t = (floorY - ro.y) / rd.y;
    vec3 h = ro + rd * t;

    if (t > 0.0 && t < nearestT) {
      nearestT = t;
      hitPos = h;
      surfaceID = 1;
    }
  }

  // --------------------------------------------------
  // Wall / floor shading
  // --------------------------------------------------

  if (surfaceID == 0) {
    // Curved wall shading.
    float angle = atan(hitPos.x, -hitPos.z);

    float u = angle * 4.2;
    float v = hitPos.y * 1.15 + 0.2;

    vec2 wp = vec2(u, v);

    col = wallPattern(wp);

    float edge = abs(angle);
    float sideDark = smoothstep(0.15, 1.35, edge);
    col *= mix(1.0, 0.42, sideDark);

    float verticalFade = smoothstep(floorY, floorY + 0.8, hitPos.y);
    col *= mix(0.65, 1.0, verticalFade);

  } else if (surfaceID == 1) {
    // Glossy dark floor.
    vec2 fp = hitPos.xz;

    vec3 floorBase = vec3(0.004, 0.006, 0.010);

    float lineX = 1.0 - smoothstep(0.015, 0.040, abs(fract(fp.x * 0.45) - 0.5));
    float lineZ = 1.0 - smoothstep(0.015, 0.040, abs(fract(fp.y * 0.45) - 0.5));
    float grid = max(lineX, lineZ);

    col = floorBase;
    col += vec3(0.012, 0.028, 0.050) * grid * 0.022;

    // Soft reflection from the neon wall.
    float angle = atan(hitPos.x, -hitPos.z);

    vec2 reflUV = vec2(
      angle * 3.2,
      -hitPos.z * 0.18 - 0.15
    );

    vec3 refl = vec3(0.0);
    refl += wallPattern(reflUV + vec2( 0.00,  0.00)) * 0.52;
    refl += wallPattern(reflUV + vec2( 0.03,  0.02)) * 0.18;
    refl += wallPattern(reflUV + vec2(-0.03,  0.02)) * 0.18;
    refl += wallPattern(reflUV + vec2( 0.00, -0.025)) * 0.12;

    float reflFade = exp(-0.065 * abs(hitPos.z)) * exp(-0.11 * hitPos.x * hitPos.x);

    // Add glossy wall reflection.
    col += refl * reflFade * 0.15;

    // Fade the floor near the horizon.
    float floorSoftFade = smoothstep(-0.95, -0.35, screen.y);
    col *= floorSoftFade;

    // Subtle darkening around the center.
    float softDark = exp(
      -1.4 * hitPos.x * hitPos.x
      -0.18 * (hitPos.z + 2.5) * (hitPos.z + 2.5)
    );

    col *= 1.0 - 0.18 * softDark;

    // --------------------------------------------------
    // Soft shadow under the glass sphere
    // --------------------------------------------------

    vec3 shadowCenter = SPHERE_CENTER;

    // Broad soft shadow.
    float broadShadow = exp(
      -3.2 * (hitPos.x - shadowCenter.x) * (hitPos.x - shadowCenter.x)
      -0.85 * (hitPos.z - shadowCenter.z) * (hitPos.z - shadowCenter.z)
    );

    // Darker contact shadow close to the floor.
    float contactShadow = exp(
      -18.0 * (hitPos.x - shadowCenter.x) * (hitPos.x - shadowCenter.x)
      -6.5  * (hitPos.z - shadowCenter.z) * (hitPos.z - shadowCenter.z)
    );

    // Slight forward stretch to ground the sphere.
    float forwardShadow = exp(
      -7.5 * (hitPos.x - shadowCenter.x) * (hitPos.x - shadowCenter.x)
      -2.0 * (hitPos.z - (shadowCenter.z + 0.10)) * (hitPos.z - (shadowCenter.z + 0.10))
    );

    float shadow = broadShadow * 0.22
                 + contactShadow * 0.38
                 + forwardShadow * 0.14;

    col *= 1.0 - shadow;

    // --------------------------------------------------
    // Soft floor reflection of the glass sphere
    // --------------------------------------------------

    // Reflection center on the floor.
    vec3 reflSphereCenter = vec3(SPHERE_CENTER.x, floorY, SPHERE_CENTER.z + 0.16);

    // Floor position relative to the reflection center.
    vec2 sphereReflPos = hitPos.xz - reflSphereCenter.xz;

    // Elliptical reflection footprint.
    vec2 q = vec2(
      sphereReflPos.x / 0.44,
      sphereReflPos.y / 0.78
    );

    // Ellipse mask.
    float orbShape = 1.0 - smoothstep(0.78, 1.10, dot(q, q));

    // Soft falloff around the reflection.
    float reflMask = exp(
      -7.5 * sphereReflPos.x * sphereReflPos.x
      -2.8 * sphereReflPos.y * sphereReflPos.y
    );

    // Inverted UV for the reflected image.
    vec2 orbPatternUV = vec2(
      q.x * 0.72,
      -q.y * 0.92 + 0.08
    );

    // Sample the wall pattern inside the reflection.
    vec3 sphereRefl = wallPattern(orbPatternUV);

    // Keep the reflection dark and subtle.
    sphereRefl = mix(vec3(0.008, 0.012, 0.018), sphereRefl, 0.45);

    // Stronger in the center, softer near the edge.
    sphereRefl *= orbShape * reflMask * 0.22;

    // Faint rim inside the reflection.
    float lowerRim = exp(
      -22.0 * q.x * q.x
      -120.0 * (q.y + 0.72) * (q.y + 0.72)
    );

    sphereRefl += vec3(0.75, 0.88, 1.0) * lowerRim * 0.04;

    // Small highlight.
    float orbHighlight = exp(
      -40.0 * (q.x + 0.10) * (q.x + 0.10)
      -65.0 * (q.y - 0.18) * (q.y - 0.18)
    );

    sphereRefl += vec3(1.0) * orbHighlight * 0.06 * orbShape;

    // Add sphere reflection to the floor.
    col += sphereRefl;

  } else {
    col = vec3(0.0);
  }

  // --------------------------------------------------
  // Ray-traced glass sphere
  // --------------------------------------------------

  vec3 rayOrigin = ro;
  vec3 rayDirection = rd;

  vec3 contribution = vec3(1.0);
  vec3 glassColor = vec3(0.0);

  bool firstRayHitGlass = false;

  const int maxLevel = 5;

  for (int level = 0; level < maxLevel; level++) {

    payload.level = level;
    payload.directLight = vec3(0.0);
    payload.nextRayOrigin = vec3(0.0);
    payload.nextRayDirection = vec3(0.0);
    payload.nextFactor = vec3(0.0);
    payload.hit = 0;

    traceRayEXT(
      topLevelAS,
      gl_RayFlagsOpaqueEXT,
      0xFFu,
      0u, 0u, 0u,
      rayOrigin,
      0.001,
      rayDirection,
      1000.0,
      0
    );

    // Store whether the primary ray hit the sphere.
    if (level == 0 && payload.hit == 1) {
      firstRayHitGlass = true;
    }

    glassColor += contribution * payload.directLight;
    contribution *= payload.nextFactor;

    if (length(payload.nextRayDirection) < 0.01) {
      break;
    }

    rayOrigin = payload.nextRayOrigin;
    rayDirection = payload.nextRayDirection;

    if (length(contribution) < 0.001) {
      break;
    }
  }

  // Only replace the procedural scene color when the primary ray hits the sphere.
  if (firstRayHitGlass) {
    col = glassColor;
    surfaceID = 2;
  }

  // --------------------------------------------------
  // Final tone adjustment
  // --------------------------------------------------

  if (surfaceID != -1) {
    float distFade = exp(-0.020 * nearestT * nearestT);
    col *= mix(0.65, 1.0, distFade);

    float vignette = 1.0 - 0.22 * dot(screen * 0.55, screen * 0.55);
    col *= vignette;
  }

  // Tone mapping and gamma correction.
  col = col / (col + vec3(1.0));
  col = pow(col, vec3(1.0 / 2.2));

  gsnSetPixel(vec4(col, 1.0));
}

/**** CLOSEST-HIT SHADER ****/
void main() {

  mat3 normalMat;
  gsnGetNormal3x3Matrix(gl_InstanceID, normalMat);

  // Normal from the procedural sphere intersection.
  vec3 N = normalize(normalMat * hit.normal);

  vec3 rayDir = normalize(gl_WorldRayDirectionEXT);
  vec3 V = normalize(-rayDir);

  vec3 hitPos = gl_WorldRayOriginEXT + gl_HitTEXT * rayDir;

  // Glass material parameters.
  vec3 baseColor = vec3(0.8, 0.8, 0.8);
  float ior = 1.5;

  float cosi = dot(V, N);

  vec3 normal = N;
  float eta = 1.0 / ior;

  if (cosi < 0.0) {
    normal = -N;
    eta = ior;
    cosi = -cosi;
  }

  vec3 F0 = vec3(0.04);
  vec3 F = fresnelSchlick(clamp(cosi, 0.0, 1.0), F0);
  float reflectAmount = clamp(F.r, 0.0, 1.0);

  vec3 reflectDir = reflect(rayDir, normal);
  vec3 refractDir = refract(rayDir, normal, eta);

  bool totalInternalReflection = length(refractDir) < 0.001;

  if (totalInternalReflection || reflectAmount > 0.65) {
    payload.nextRayDirection = normalize(reflectDir);
    payload.nextFactor = vec3(0.9);
  } else {
    payload.nextRayDirection = normalize(refractDir);
    payload.nextFactor = baseColor;
  }

  payload.nextRayOrigin = hitPos + payload.nextRayDirection * 0.01;

  // The sphere itself emits no light; color comes from the next ray.
  payload.directLight = vec3(0.0);
  payload.hit = 1;
}

/**** MISS SHADER ****/
void main() { /**** MISS SHADER ****/

  vec3 dir = normalize(gl_WorldRayDirectionEXT);

  // Convert ray direction to procedural background coordinates.
  float angle = atan(dir.x, -dir.z);

  vec2 uv = vec2(
    angle * 3.2,
    dir.y * 1.5 + 0.15
  );

  vec3 env = wallPattern(uv);

  // Small base brightness for the dark background.
  env += vec3(0.002, 0.004, 0.008);

  payload.directLight = env;

  payload.nextRayOrigin = vec3(0.0);
  payload.nextRayDirection = vec3(0.0);
  payload.nextFactor = vec3(0.0);
  payload.hit = 0;
}

/**** INTERSECTION SHADER ****/
void main() { /**** INTERSECTION SHADER ****/

  // Sphere intersection inside the selected bounding box.
  vec3 aabbMin;
  vec3 aabbMax;
  gsnGetIntersectionBox(gl_InstanceID, gl_PrimitiveID, aabbMin, aabbMax);

  vec3 center = (aabbMin + aabbMax) / 2.0;
  float radius = length(aabbMax.x - aabbMin.x) / 2.0;

  vec3 b = gl_ObjectRayOriginEXT - center;
  vec3 r = gl_ObjectRayDirectionEXT;

  float rr = dot(r, r);
  float p = 2.0 * dot(r, b) / rr;
  float q = (dot(b, b) - radius * radius) / rr;

  float d = (p * p / 4.0) - q;

  if (d < 0.0) return;

  float t1 = -p / 2.0 - sqrt(d);
  float t2 = -p / 2.0 + sqrt(d);

  float t = t2;
  uint hitKind = 2u;

  if (t1 < t2 && t1 >= gl_RayTminEXT && t1 <= gl_RayTmaxEXT) {
    t = t1;
    hitKind = 1u;
  }

  if (t >= gl_RayTminEXT && t <= gl_RayTmaxEXT) {
    vec3 hitPoint = gl_ObjectRayOriginEXT + t * gl_ObjectRayDirectionEXT;
    hit.normal = normalize(hitPoint - center);
    reportIntersectionEXT(t, hitKind);
  }
}
