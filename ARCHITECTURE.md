# Yemen Wedding AI — Production Architecture
## v2.0.0 | Play Store Ready AI Wedding Platform

---

## 1. SYSTEM ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT LAYER (Capacitor)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   React/    │  │  Native     │  │  Offline Cache (SW)     │ │
│  │   Next.js   │  │  Plugins    │  │  IndexedDB + Cache API  │ │
│  │   (PWA)     │  │  Camera/FS  │  │  50MB Image Buffer      │ │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘ │
│         │                │                       │               │
│  ┌──────▼────────────────▼───────────────────────▼─────────────┐│
│  │              State Management (Zustand)                     ││
│  │  Auth | Gallery | Generation Queue | Settings | Offline     ││
│  └─────────────────────────────────────────────────────────────┘│
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS / WebSocket
┌───────────────────────────▼─────────────────────────────────────┐
│                   API GATEWAY (Cloudflare/AWS)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ Rate Limit  │  │   Auth      │  │  Request Routing        │ │
│  │ 100 req/min │  │  JWT/OAuth2 │  │  /generate /gallery    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│              SERVERLESS FUNCTIONS (Vercel/Cloudflare)           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  /generate  │  │  /enhance   │  │  /webhook (replicate)  │ │
│  │  /gallery   │  │  /face-fix  │  │  /stripe/webhook       │ │
│  │  /auth      │  │  /upscale   │  │  /export               │ │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘ │
└─────────┼────────────────┼────────────────────┼────────────────┘
          │                │                    │
┌─────────▼────────────────▼────────────────────▼────────────────┐
│              AI PROVIDER ABSTRACTION LAYER                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  Replicate  │  │ Leonardo AI │  │  Stability AI          │ │
│  │  SDXL +     │  │  Phoenix    │  │  SD3 / SDXL            │ │
│  │  Custom     │  │  Alchemy    │  │  Image Core            │ │
│  │  LoRA       │  │  v2         │  │                        │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────────────────────────────────────────────────┐
│              POST-PROCESSING PIPELINE                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   GFPGAN    │  │ CodeFormer  │  │  Real-ESRGAN (4x)      │ │
│  │  Face Det.  │  │  Face Rest. │  │  Upscaling             │ │
│  │  + Restore  │  │  + Enhance  │  │  + Sharpening          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└───────────────────────────┬────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────┐
│              DATA & STORAGE LAYER                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  Supabase   │  │  Cloudflare │  │  Stripe                │ │
│  │  PostgreSQL │  │  R2 / S3    │  │  Payments              │ │
│  │  Auth + RT  │  │  CDN + Img  │  │  Subscriptions         │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. AI PROMPT ENGINEERING — YEMENI WEDDING

### 2.1 Base Prompt Templates

```typescript
// Sanaani Traditional Bride
const sanaaniBridePrompt = `
Ultra-realistic portrait of a Yemeni bride in traditional Sanaani wedding attire,
wearing elaborate silver jewelry (Al-Zannah headdress, Al-Massar forehead chain, 
Al-Raash silver necklace with coral and amber), intricate henna patterns on hands,
white jasmine flowers in hair, wearing a black velvet dress with gold embroidery,
authentic Arabic facial features, warm olive skin tone, dark expressive eyes,
cinematic golden hour lighting, shallow depth of field, professional wedding 
photography, 8K resolution, photorealistic, shot on Canon EOS R5
`;

// Hadrami Luxury Bride
const hadramiBridePrompt = `
Luxurious portrait of a Hadrami Yemeni bride in traditional Dawani dress,
heavy silver belt (Al-Hizam Al-Fiddi), amber and coral jewelry layers,
intricate beadwork on veil, authentic Hadramaut architectural background,
mountain landscape of Wadi Hadramaut, dramatic cinematic lighting,
professional fashion photography, ultra-detailed fabric textures,
8K, photorealistic, shot on Hasselblad H6D
`;

// Yemeni Groom with Jambiya
const groomPrompt = `
Distinguished Yemeni groom in traditional wedding attire, wearing ornate Jambiya 
(curved dagger) with silver hilt and sheath, white Thobe with gold trim,
embroidered vest (Al-Sidra), traditional turban (Al-Mahrama) with gold agal,
authentic Arabic facial features with well-groomed beard, confident posture,
luxury palace interior background with Yemeni geometric patterns,
cinematic portrait lighting, 8K photorealistic, professional photography
`;

// Royal Yemeni Couple
const royalCouplePrompt = `
Majestic Yemeni bride and groom in royal wedding attire, bride in gold-embroidered
black velvet Thobe with crown headdress and layers of silver/gold jewelry,
groom in white ceremonial Thobe with ornate Jambiya and gold-embroidered vest,
standing in Old Sana'a architecture background with iconic tower houses,
golden sunset lighting, romantic pose, cinematic composition,
professional wedding photography, 8K ultra-detailed, photorealistic
`;
```

### 2.2 Negative Prompts

```typescript
const negativePrompt = `
low quality, blurry, distorted face, deformed hands, extra fingers,
mutated, ugly, duplicate, watermark, signature, text, logo, cropped,
worst quality, low resolution, cartoon, anime, painting, illustration,
western features, light skin, blue eyes, blonde hair, modern clothing,
western wedding dress, suit, tie, inappropriate content
`;
```

### 2.3 LoRA Weights Configuration

```typescript
interface LoRAConfig {
  model: string;
  weight: number;
  triggerWord: string;
}

const loraModels: Record<string, LoRAConfig[]> = {
  sanaani: [
    { model: "yemeni-sanaani-dress-v2", weight: 0.85, triggerWord: "sanaani_dress" },
    { model: "arabic-silver-jewelry-v1", weight: 0.75, triggerWord: "arabic_silver" },
    { model: "yemeni-henna-patterns-v1", weight: 0.65, triggerWord: "yemeni_henna" }
  ],
  hadrami: [
    { model: "hadrami-dawani-dress-v2", weight: 0.9, triggerWord: "hadrami_dawani" },
    { model: "yemeni-amber-jewelry-v1", weight: 0.7, triggerWord: "yemeni_amber" },
    { model: "arabian-desert-bg-v1", weight: 0.5, triggerWord: "arabian_desert" }
  ],
  royal: [
    { model: "royal-yemeni-wedding-v3", weight: 0.9, triggerWord: "royal_yemeni" },
    { model: "gold-embroidery-luxury-v2", weight: 0.8, triggerWord: "gold_embroidery" },
    { model: "old-sanaa-architecture-v1", weight: 0.6, triggerWord: "old_sanaa" }
  ],
  groom: [
    { model: "yemeni-jambiya-dagger-v2", weight: 0.85, triggerWord: "yemeni_jambiya" },
    { model: "arabic-male-attire-v1", weight: 0.75, triggerWord: "arabic_male" }
  ]
};
```

---

## 3. API INTEGRATIONS

### 3.1 Replicate (Primary)

```typescript
// api/generate/replicate.ts
import Replicate from "replicate";

const replicate = new Replicate({
  auth: process.env.REPLICATE_API_TOKEN,
});

export async function generateWithReplicate(params: GenerationParams) {
  const { style, character, background, details, quality, lighting } = params;

  const prompt = buildPrompt({ style, character, background, details, lighting });
  const loras = loraModels[style] || [];

  const output = await replicate.run(
    "stability-ai/sdxl:39ed52f2a78e934b3ba6e2a89f5b1c712de7dfea535525255b1aa35c5565e08b",
    {
      input: {
        prompt: prompt,
        negative_prompt: negativePrompt,
        width: quality === "4K" ? 1536 : quality === "2K" ? 1216 : 1024,
        height: quality === "4K" ? 2048 : quality === "2K" ? 1624 : 1344,
        num_outputs: 1,
        scheduler: "K_EULER",
        num_inference_steps: 50,
        guidance_scale: 7.5,
        refine: "expert_ensemble_refiner",
        high_noise_frac: 0.8,
        lora_scale: 0.8,
        // Apply LoRA models
        ...(loras.length > 0 && {
          apply_lora: true,
          lora_weights: loras.map(l => l.model).join(","),
          lora_scales: loras.map(l => l.weight).join(","),
        })
      }
    }
  );

  return output;
}
```

### 3.2 Leonardo AI (Secondary)

```typescript
// api/generate/leonardo.ts
const LEONARDO_API = "https://cloud.leonardo.ai/api/rest/v1";

export async function generateWithLeonardo(params: GenerationParams) {
  const response = await fetch(`${LEONARDO_API}/generations`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.LEONARDO_API_KEY}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      prompt: buildPrompt(params),
      negative_prompt: negativePrompt,
      modelId: "6bef9f1b-29cb-40c7-b9df-32b51c1f934d", // Phoenix
      width: 1024,
      height: 1536,
      num_images: 1,
      guidance_scale: 7,
      alchemy: true,
      photoReal: true,
      photoRealStrength: 0.55,
      presetStyle: "CINEMATIC",
      // Custom LoRA
      lora_models: loraModels[params.style]?.map(l => ({
        lora_model_id: l.model,
        strength: l.weight
      })) || []
    })
  });

  return response.json();
}
```

### 3.3 Stability AI (Tertiary)

```typescript
// api/generate/stability.ts
const STABILITY_API = "https://api.stability.ai/v2beta/stable-image/generate/sd3";

export async function generateWithStability(params: GenerationParams) {
  const formData = new FormData();
  formData.append("prompt", buildPrompt(params));
  formData.append("negative_prompt", negativePrompt);
  formData.append("aspect_ratio", "2:3");
  formData.append("model", "sd3-medium");
  formData.append("output_format", "png");
  formData.append("cfg_scale", "7");

  const response = await fetch(STABILITY_API, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.STABILITY_API_KEY}`,
      "Accept": "image/*"
    },
    body: formData
  });

  return response.blob();
}
```

### 3.4 Face Enhancement Pipeline

```typescript
// api/enhance/face.ts
export async function enhanceFace(imageUrl: string) {
  // Step 1: GFPGAN for face restoration
  const gfpganOutput = await replicate.run(
    "tencentarc/gfpgan:9283608cc6b7be6b65a8e44983db012355fde4132009bf99d976b2f0896856a3",
    { input: { img: imageUrl, version: "v1.4", scale: 2 } }
  );

  // Step 2: CodeFormer for face enhancement
  const codeformerOutput = await replicate.run(
    "sczhou/codeformer:7de2ea26c616d5bf2245ad0d5e24f0ff9a6204578a5c876db53142edd70263e4",
    { 
      input: { 
        image: gfpganOutput,
        codeformer_fidelity: 0.7,
        background_enhance: true,
        face_upsample: true,
        upscale: 2
      } 
    }
  );

  return codeformerOutput;
}
```

---

## 4. SUBSCRIPTION TIERS

| Feature | Free | Premium $9.99/mo | VIP Studio $29.99/mo |
|---------|------|------------------|---------------------|
| Generations/month | 5 | 100 | Unlimited |
| Max Resolution | HD (1024x1344) | 4K (1536x2048) | 8K (3072x4096) |
| AI Providers | Replicate only | All 3 providers | All + Priority |
| LoRA Models | 3 basic | All + Custom | Private LoRA training |
| Face Enhancement | Basic | GFPGAN + CodeFormer | + Real-ESRGAN 4x |
| Watermark | Yes | No | No |
| Video Generation | No | No | Yes (AI Video) |
| API Access | No | No | Yes |
| Support | Community | Email | 24/7 Priority |
| Export Formats | JPG | JPG, PNG, WebP | + TIFF, RAW |
| Cloud Storage | 50MB | 5GB | Unlimited |

---

## 5. DATABASE SCHEMA (Supabase)

```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  avatar_url TEXT,
  tier TEXT DEFAULT 'free' CHECK (tier IN ('free', 'premium', 'vip')),
  generations_used INTEGER DEFAULT 0,
  generations_limit INTEGER DEFAULT 5,
  storage_used_mb INTEGER DEFAULT 0,
  storage_limit_mb INTEGER DEFAULT 50,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Generations
CREATE TABLE generations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  prompt TEXT NOT NULL,
  negative_prompt TEXT,
  image_url TEXT NOT NULL,
  thumbnail_url TEXT,
  style TEXT,
  provider TEXT,
  quality TEXT,
  width INTEGER,
  height INTEGER,
  seed BIGINT,
  is_favorite BOOLEAN DEFAULT false,
  is_watermarked BOOLEAN DEFAULT true,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Subscriptions
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  stripe_subscription_id TEXT,
  stripe_customer_id TEXT,
  tier TEXT NOT NULL,
  status TEXT DEFAULT 'active',
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  cancel_at_period_end BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- LoRA Models
CREATE TABLE lora_models (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  description TEXT,
  trigger_word TEXT,
  replicate_model_id TEXT,
  leonardo_model_id TEXT,
  category TEXT,
  is_premium BOOLEAN DEFAULT false,
  is_custom BOOLEAN DEFAULT false,
  user_id UUID REFERENCES users(id),
  download_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Prompt Templates
CREATE TABLE prompt_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  style TEXT NOT NULL,
  character TEXT,
  background TEXT,
  details TEXT[],
  lighting TEXT,
  prompt_text TEXT NOT NULL,
  negative_prompt TEXT,
  is_system BOOLEAN DEFAULT false,
  user_id UUID REFERENCES users(id),
  usage_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- RLS Policies
ALTER TABLE generations ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can only see their own generations"
  ON generations FOR ALL USING (auth.uid() = user_id);

ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can only see their own subscriptions"
  ON subscriptions FOR ALL USING (auth.uid() = user_id);
```

---

## 6. STRIPE INTEGRATION

```typescript
// lib/stripe.ts
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2024-04-10",
});

const PRICE_IDS = {
  premium_monthly: "price_1P...",
  premium_yearly: "price_1P...",
  vip_monthly: "price_1P...",
  vip_yearly: "price_1P...",
};

export async function createCheckoutSession(userId: string, tier: string) {
  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    line_items: [{
      price: PRICE_IDS[`${tier}_monthly`],
      quantity: 1,
    }],
    mode: "subscription",
    success_url: `${process.env.APP_URL}/subscription/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.APP_URL}/subscription/cancel`,
    metadata: { userId, tier },
    subscription_data: {
      metadata: { userId }
    }
  });

  return session;
}

// Webhook handler
export async function handleWebhook(req: Request) {
  const sig = req.headers.get("stripe-signature");
  const event = stripe.webhooks.constructEvent(
    await req.text(),
    sig!,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  switch (event.type) {
    case "checkout.session.completed":
      await activateSubscription(event.data.object);
      break;
    case "invoice.payment_succeeded":
      await extendSubscription(event.data.object);
      break;
    case "customer.subscription.deleted":
      await downgradeToFree(event.data.object);
      break;
  }
}
```

---

## 7. PERFORMANCE OPTIMIZATIONS

```typescript
// Image optimization pipeline
const imagePipeline = {
  // 1. Lazy loading with IntersectionObserver
  lazyLoad: (img: HTMLImageElement) => {
    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          img.src = img.dataset.src!;
          observer.unobserve(img);
        }
      });
    }, { rootMargin: "200px" });
    observer.observe(img);
  },

  // 2. Progressive loading (blur-up)
  progressiveLoad: async (src: string) => {
    // Load tiny thumbnail first
    const thumb = await fetch(`${src}?w=20&q=10`);
    // Then load full image
    const full = await fetch(`${src}?w=800&q=85`);
    return { thumb, full };
  },

  // 3. WebP conversion with fallback
  formatImage: (src: string) => {
    const supportsWebP = document.createElement('canvas')
      .toDataURL('image/webp').indexOf('data:image/webp') === 0;
    return supportsWebP ? `${src}.webp` : `${src}.jpg`;
  },

  // 4. Responsive srcset
  generateSrcSet: (baseUrl: string) => {
    return [320, 640, 960, 1280, 1920]
      .map(w => `${baseUrl}?w=${w}&q=85 ${w}w`)
      .join(', ');
  }
};

// Service Worker for offline caching
// sw.js
const CACHE_NAME = 'yemen-wedding-ai-v2';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/static/js/main.js',
  '/static/css/main.css',
  '/fonts/cairo.woff2',
  '/fonts/tajawal.woff2'
];

self.addEventListener('install', (e) => {
  e.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(STATIC_ASSETS))
  );
});

self.addEventListener('fetch', (e) => {
  e.respondWith(
    caches.match(e.request).then(response => {
      return response || fetch(e.request).then(fetchResponse => {
        // Cache images for offline viewing
        if (e.request.url.match(/\.(jpg|png|webp)$/)) {
          const clone = fetchResponse.clone();
          caches.open(CACHE_NAME).then(cache => cache.put(e.request, clone));
        }
        return fetchResponse;
      });
    })
  );
});
```

---

## 8. SECURITY CHECKLIST

- [x] API keys stored in environment variables (never client-side)
- [x] JWT authentication with refresh tokens
- [x] Rate limiting: 100 req/min per user, 10 req/min for free tier
- [x] Input sanitization for all prompts (XSS prevention)
- [x] Content moderation via Replicate's safety checker
- [x] HTTPS-only with HSTS headers
- [x] CORS configured for app domain only
- [x] Stripe webhook signature verification
- [x] Database RLS policies enabled
- [x] Image upload validation (type, size, dimensions)
- [x] CSRF tokens for state-changing operations
- [x] Audit logging for all generation requests

---

## 9. DEPLOYMENT CHECKLIST

### Play Store Release
- [ ] Capacitor build for Android
- [ ] App icon (512x512) + adaptive icons
- [ ] Feature graphic (1024x500)
- [ ] Screenshots (phone + tablet)
- [ ] Privacy policy URL
- [ ] Content rating questionnaire
- [ ] Target API level 34+
- [ ] Signed APK/AAB
- [ ] Play Console listing in Arabic + English

### Backend Deployment
- [ ] Vercel Pro for serverless functions
- [ ] Supabase project with RLS
- [ ] Cloudflare R2 for image storage
- [ ] Stripe live mode activated
- [ ] Replicate production API key
- [ ] Leonardo AI production key
- [ ] Custom domain + SSL
- [ ] Monitoring (Sentry + LogRocket)

---

## 10. FILE STRUCTURE

```
yemen-wedding-ai/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   └── layout.tsx
│   │   ├── (main)/
│   │   │   ├── page.tsx              # Home
│   │   │   ├── generate/page.tsx     # AI Generation
│   │   │   ├── gallery/page.tsx      # Gallery
│   │   │   ├── result/page.tsx       # Result View
│   │   │   ├── subscription/page.tsx # Plans
│   │   │   ├── profile/page.tsx      # User Profile
│   │   │   ├── settings/page.tsx     # Settings
│   │   │   └── layout.tsx
│   │   ├── api/
│   │   │   ├── generate/
│   │   │   │   ├── replicate/route.ts
│   │   │   │   ├── leonardo/route.ts
│   │   │   │   └── stability/route.ts
│   │   │   ├── enhance/
│   │   │   │   ├── face/route.ts
│   │   │   │   └── upscale/route.ts
│   │   │   ├── auth/
│   │   │   │   └── [...nextauth]/route.ts
│   │   │   ├── stripe/
│   │   │   │   └── webhook/route.ts
│   │   │   └── gallery/route.ts
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/                       # Reusable UI
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Chip.tsx
│   │   │   ├── Toggle.tsx
│   │   │   ├── Modal.tsx
│   │   │   └── Toast.tsx
│   │   ├── generation/
│   │   │   ├── StyleSelector.tsx
│   │   │   ├── ProviderPicker.tsx
│   │   │   ├── OptionChips.tsx
│   │   │   ├── GenerationLoader.tsx
│   │   │   └── BeforeAfterSlider.tsx
│   │   ├── gallery/
│   │   │   ├── MasonryGrid.tsx
│   │   │   ├── GalleryCard.tsx
│   │   │   └── FullImageView.tsx
│   │   ├── subscription/
│   │   │   ├── PlanCard.tsx
│   │   │   └── CheckoutButton.tsx
│   │   └── layout/
│   │       ├── BottomNav.tsx
│   │       ├── AppHeader.tsx
│   │       └── SplashScreen.tsx
│   ├── lib/
│   │   ├── supabase.ts
│   │   ├── stripe.ts
│   │   ├── replicate.ts
│   │   ├── leonardo.ts
│   │   ├── prompts.ts
│   │   ├── lora.ts
│   │   └── utils.ts
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useGeneration.ts
│   │   ├── useGallery.ts
│   │   ├── useSubscription.ts
│   │   └── useOffline.ts
│   ├── stores/
│   │   ├── authStore.ts
│   │   ├── generationStore.ts
│   │   ├── galleryStore.ts
│   │   └── settingsStore.ts
│   ├── types/
│   │   ├── generation.ts
│   │   ├── user.ts
│   │   ├── subscription.ts
│   │   └── api.ts
│   └── styles/
│       └── globals.css
├── public/
│   ├── icons/
│   ├── images/
│   └── fonts/
├── capacitor/
│   ├── android/
│   └── ios/
├── supabase/
│   ├── migrations/
│   └── seed.sql
├── .env.local
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
├── package.json
└── README.md
```

---

**Built with love for Yemeni heritage 🇾🇪**
**Yemen Wedding AI v2.0.0 Production**
