# Hi.Events — Production Setup & Running Costs (real usage, Stripe live) — prices in BRL

> Researched & verified 2026-07-17. Fork: fsspx-br/Hi.Events (synced with upstream v1.11.0-beta).
> **FX**: spot US$1 ≈ R$5.10. USD purchases pay **IOF 3.5%** (2028 phase-out suspended — Decreto 12.499/2025) + issuer spread → effective ≈ R$5.45/US$. **With a BRL-billed VPS, the only USD line left is SES (~R$2–11/mo) — the stack is essentially de-dollarized.**
> Goal: reliable ticket sales with Stripe. Free-tier gambles excluded from the recommendation.

## 1. What must run (from repo, `docker/all-in-one/`)

One Docker host runs everything — admin dashboard (`/manage`), public event pages + checkout, API, queue workers, Postgres 17, Redis 7. File uploads on local disk by default (no S3 needed).

**Why reliability matters with Stripe specifically:**
- Stripe confirms payments via **webhooks to your server** — box down = orders stuck unconfirmed during a sale.
- Checkout traffic spikes when an event opens — needs RAM/CPU headroom.
- Order/ticket emails go through queue workers — must be running, always.

## 2. One-time / annual

| Item | Cost | Notes |
|---|---|---|
| Domain `.com.br` | **R$40/year** | direct at registro.br (BRL, no FX) |
| TLS | R$0 | Let's Encrypt or Cloudflare proxy |
| AWS account + SES production access | R$0 | ~R$1,000 credits for new accounts; request sandbox exit BEFORE launch |
| SPF + DKIM + DMARC DNS records | R$0 | mandatory for deliverability |
| Stripe account (BR) | R$0 | CNPJ **or CPF** (empresário individual) — see §4 |
| Hostinger 24-month prepay (if promo route) | ~R$1,032 upfront | = R$42.99 × 24; what unlocks the promo price |

## 3. VPS — verified prices (checked 2026-07-17 on providers' sites)

### BRL-billed providers (no IOF, no FX float)
| Provider / plan | Spec | Promo (24-mo contract) | Renewal / no-contract | Backups |
|---|---|---|---|---|
| **Hostinger KVM2** ⭐ | 2 vCPU / **8 GB** / 100 GB NVMe | **R$42.99** | R$77.99 renewal · R$108.99 monthly | **weekly incluídos** + snapshots |
| Hostinger KVM1 | 1 vCPU / 4 GB / 50 GB NVMe | R$29.99 | R$59.99 renewal | weekly incluídos |
| Locaweb VPS 4GB | 2 vCPU / 4 GB / 70 GB SSD | R$53.90 | R$72.90 renewal | 1 snapshot |
| KingHost VPS 4GB | 2 vCPU / 4 GB / 70 GB SSD | R$89.90 | — | não especificado |

### USD-billed reference (converted at R$5.45)
| Provider | Spec | Effective BRL |
|---|---|---|
| AWS Lightsail SP 4 GB | 2 vCPU / 4 GB / 80 GB | ~R$130 (US$24) + 20% backup |
| AWS Lightsail SP 2 GB | 2 vCPU / 2 GB / 60 GB | ~R$65 (US$12) |

**Winner: Hostinger KVM2** — double the RAM of everything else at the price, NVMe, weekly backups included, BRL billing. Caveats:
- Promo needs **24-month prepay (~R$1,032)**; renewal R$77.99 — still ~40% below Lightsail. No-contract monthly R$108.99 ≈ Lightsail anyway → the prepay IS the discount.
- Confirm **São Paulo datacenter** at checkout (site says "América do Sul"; SP widely documented — verify before paying).
- Consumer-grade support/SLA, not AWS. Mitigations below (Cloudflare, off-site dumps, restore drill) cover the realistic failure modes.
- Weekly image backups ≠ enough for order data → nightly `pg_dump` off-site stays mandatory.
- Your pasted prices were stale: KingHost "R$29.90" is actually R$89.90 today; Locaweb "R$69.90" is the renewal (R$72.90), promo R$53.90.

## 4. Monthly costs — recommended setup

| Item | Monthly (BRL) |
|---|---|
| Hostinger KVM2 (promo, prepaid 24 mo) | R$42.99 |
| Off-site DB backup (`pg_dump` → B2/R2 free tier) | R$0–5 |
| Amazon SES `sa-east-1` (~R$0.55/1k emails; ~3/order) | R$2–11 |
| Cloudflare Free + UptimeRobot + Sentry | R$0 |
| **Total** | **~R$46–59/mo** |

After the 24-month promo: ~R$81–94/mo at renewal price. No-contract route: ~R$111–125/mo (at that point Lightsail R$135–175 with AWS-grade infra is worth considering).

### Optional hardening (when revenue justifies)
| Item | Monthly | When |
|---|---|---|
| Managed Postgres (DO/Neon) | +R$82+ (USD) | lost DB = unacceptable |
| Warm-standby VPS (Hostinger KVM1) | +R$30–60 | downtime during a sale > R$400/yr |

### Not for production
- **Oracle Cloud free tier** — halved Jun/2026, capacity scarce, reclaimable. **Free staging box** only.
- **Home hosting** — residential IP/CGNAT breaks Stripe webhooks + email reputation.
- **Hetzner** (~R$24) — no BR region (~200 ms); non-BR audiences only.

## 5. Variable costs

### Email (SES) — uniform pricing in all regions (sa-east-1 confirmed)
**~R$0.55 per 1,000 emails** (US$0.10). 1,000 tickets/mo ≈ 3,000 emails ≈ **R$1.70/mo**. Free tier 3k/mo for 12 months. Keep bounce <2%.

### Stripe Brazil — the dominant cost (native BRL)
| Method | Fee | 1,000 × R$50 tickets |
|---|---|---|
| Domestic credit card | **3.99% + R$0.39** | **~R$2,390/mo** |
| International card | +2% | — |
| Pix via Stripe | 1.19% — **invite-only** (60-day processing history + good standing) | ~R$595/mo |
| Manual Pix (Hi.Events offline-payment mode) | 0% + labor | R$0 |

**Account opening (verified):** CPF accepted (empresário individual). PF↔PJ **immutable after verification** — switching to CNPJ later = new Stripe account; decide before opening. Payout account: BRL, Brazilian bank, same CPF/CNPJ. Self-hosted = zero platform fee.

## 6. Production reliability checklist (setup week)

- [ ] VPS provisioned (confirm SP datacenter at checkout), Docker stack up, domain + TLS live
- [ ] SES out of sandbox; SPF/DKIM/DMARC verified; test order email inboxes on Gmail + Outlook
- [ ] Stripe **live** keys + `STRIPE_WEBHOOK_SECRET`; webhook green in dashboard; end-to-end test purchase (buy → webhook → paid → ticket email)
- [ ] Weekly provider backup confirmed ON; nightly `pg_dump` cron to B2/R2 verified
- [ ] **Restore drill**: fresh VPS from backup + dump, app boots — know recovery time before you need it
- [ ] UptimeRobot: **no dedicated health endpoint exists** (verified — no `/up`/`/health` route). Monitor **homepage + one public event page** (exercises SSR + API + DB)
- [ ] Sentry DSN configured
- [ ] Queue workers + scheduler survive reboot (`docker compose restart` test)

## 7. Ongoing ops (time, not money)

- **Upgrades**: snapshot first → pinned image tag → `php artisan migrate` → test checkout. Rehearse on staging (Oracle free / local Docker). ~Monthly.
- **Watch weekly**: Stripe webhook failures (`webhook_logs` table + dashboard), SES bounce rate, disk space.
- **Quarterly**: restore drill; creds/DNS documented somewhere safe.
- **Calendar**: Hostinger renewal date (month 24) — renegotiate or migrate; price jumps R$42.99 → R$77.99.
- **Monthly, during fork sync**: check the Pix/upstream watch list (§10) — PR #1038 merging changes our Pix strategy.

## 8. Bottom line (BRL)

| | Monthly | Upfront/annual |
|---|---|---|
| **Recommended (Hostinger KVM2 promo)** | **~R$46–59** | R$1,032 prepay (24 mo) + R$40/yr domain |
| Same, after promo (renewal) | ~R$81–94 | R$40/yr domain |
| No-contract (Hostinger monthly) | ~R$111–125 | R$40/yr domain |
| AWS Lightsail route (no lock-in, AWS-grade) | ~R$135–175 | R$40/yr domain |
| Staging | R$0 | — |

Plus ~4.4–4.8% of card revenue to Stripe (or 1.19% Pix when invited / 0% manual Pix).
**Rule of thumb: infra now costs about ONE R$50 ticket per month.** The money conversation is Pix access and deliverability, not servers.

---

## 9. Resumo não técnico (para compartilhar)

*Esta seção é independente do resto do documento — pode copiar e mandar como está.*

Vamos vender ingressos pelo nosso próprio site (sistema Hi.Events, gratuito e de código aberto). Os custos são dois: uma **mensalidade fixa pequena** pra manter o site no ar, e uma **taxa por venda** que a Stripe (a "maquininha" online) cobra.

### Custo fixo mensal

| Item | Custo/mês |
|---|---|
| Servidor (hospeda o site, as vendas e o painel) | R$ 43,00 |
| Envio de e-mails (ingressos e confirmações) | ~R$ 2,00 |
| Domínio — o endereço do site (R$ 40/ano) | R$ 3,33 |
| Backups, monitoramento e segurança | R$ 0–5 |
| **Total** | **~R$ 55/mês** |

*Obs.: o servidor pede pagamento antecipado de 24 meses (~R$ 1.032) para garantir esse preço promocional.*

### Taxas da Stripe (por venda)

| Forma de pagamento | Taxa |
|---|---|
| Cartão de crédito nacional | 3,99% + R$ 0,39 |
| Cartão internacional | 5,99% + R$ 0,39 |
| Pix pela Stripe (precisa de liberação, não vem de início) | 1,19% |
| Pix manual (direto na nossa conta, confirmação na mão) | R$ 0 |

### Simulação: ingresso de R$ 30

| Forma de pagamento | Stripe leva | Cai na conta |
|---|---|---|
| Cartão nacional | R$ 1,59 | **R$ 28,41** |
| Pix pela Stripe | R$ 0,36 | R$ 29,64 |
| Pix manual | R$ 0,00 | R$ 30,00 |

E descontando também o custo fixo do mês (R$ 55), vendendo tudo no cartão:

| Ingressos vendidos | Faturamento | Stripe leva | Sistema | **Sobra no bolso** |
|---|---|---|---|---|
| 50 | R$ 1.500 | R$ 79 | R$ 55 | **R$ 1.366** |
| 100 | R$ 3.000 | R$ 159 | R$ 55 | **R$ 2.786** |
| 300 | R$ 9.000 | R$ 476 | R$ 55 | **R$ 8.469** |
| 500 | R$ 15.000 | R$ 794 | R$ 55 | **R$ 14.152** |

### Quanto cobrar para ter X de lucro?

Fórmula (pagamento no cartão — o pior caso; Pix só melhora):

```
Preço do ingresso = (Lucro desejado + Custos + 0,39 × N) ÷ (0,9601 × N)
```

- **N** = quantos ingressos espera vender
- **Custos** = sistema (R$ 55/mês) + custos do evento (local, som, comes e bebes…)
- 0,9601 e 0,39 vêm da taxa do cartão (3,99% + R$ 0,39)

**Exemplo:** queremos R$ 5.000 de lucro num evento com R$ 3.000 de custos (+ R$ 55 do sistema), esperando 300 ingressos:

```
Preço = (5.000 + 3.055 + 0,39 × 300) ÷ (0,9601 × 300)
      = 8.172 ÷ 288,03
      = R$ 28,37  →  cobrando R$ 30, sobra folga (lucro ≈ R$ 5.470)
```

Conferindo na direção contrária (quanto lucro um preço dá):

```
Lucro = N × (0,9601 × Preço − 0,39) − Custos
```

Versões da fórmula para as outras formas de pagamento:
- **Pix pela Stripe**: `Preço = (Lucro + Custos) ÷ (0,9881 × N)`
- **Pix manual**: `Preço = (Lucro + Custos) ÷ N`

Regra de bolso: **o sistema inteiro custa o equivalente a ~2 ingressos de R$ 30 por mês** — o que pesa de verdade é a taxa do cartão, então quanto mais gente pagar no Pix, mais sobra.

---

## 10. Pix automation — upstream watch list

**State of the world (verified 2026-07-17):** nobody has built Pix for Hi.Events yet — zero upstream issues/discussions (EN + PT), no diverged forks (BR-named forks are 0 commits ahead), no code or blog posts anywhere. We'd be first. Until then: **manual Pix via the built-in offline-payment mode (0%) + Stripe cards.**

### Watch these — check monthly during fork sync

| What | Why it matters | Trigger → action |
|---|---|---|
| [PR #1038 — Feature/razorpay](https://github.com/HiEventsDev/hi.events/pull/1038) (community, 93 files, open Feb/2026, last update Jun/2026) | Introduces the **multi-gateway abstraction**: extends `PaymentProviders` enum, gateway webhook actions, gateway-agnostic payment/refund status. A leftover `RazorpayOrderDomainObject` already leaked into develop | **MERGED → revisit native Pix provider** (Option 2): this PR is the file-by-file blueprint; a Mercado Pago/Efí Pix provider becomes a clean follow-on and a strong upstream PR (Pix = 46% of BR payments) |
| [Issue #458 — Support additional payment methods](https://github.com/HiEventsDev/hi.events/issues/458) (on official roadmap) | Maintainer signal on multi-gateway direction | Maintainer comments → gauge appetite for a contributed Pix provider |
| Stripe Pix invite for our account | 1.19%, fully automatic | Checkout already uses `automatic_payment_methods` (`StripePaymentIntentCreationService.php`) → when invited, **enable in Stripe dashboard, zero code** |

### Auto-Pix option ladder (when manual matching starts to hurt)

1. **n8n bridge, no fork changes (~0.8–1%)** — `order.created` webhook → n8n creates dynamic QR at PSP (OpenPix/Mercado Pago/Efí, `txid` = order ref) → buyer pays → PSP webhook → n8n calls `POST /events/{id}/orders/{id}/mark-as-paid` (endpoint verified, exists) → ticket sent automatically.
2. **Native Pix provider in the fork (0–1%)** — build after PR #1038 merges, using it as the template; banks like Inter PJ offer free Pix APIs (verify terms). Aim to upstream = maintenance moves upstream.
3. **Stripe Pix (1.19%)** — zero work, wait for invite; needs 60 days of processing history + good standing.

## Verification log (2026-07-17)

| Item | Verdict |
|---|---|
| BR VPS prices (user list was stale) | Hostinger KVM2 8GB **R$42.99** promo/24mo (renewal R$77.99, monthly R$108.99), weekly backups incl.; Locaweb 4GB R$53.90 promo/R$72.90 renewal; KingHost 4GB **R$89.90** (not R$29.90) |
| Lightsail SP | 2GB US$12, 4GB US$24 exact |
| IOF 2026 | **3.5% fixed** — phase-out suspended (Decreto 12.499/2025) |
| Stripe BR onboarding | CPF ok (empresário individual); PF/PJ immutable; Pix invite-only |
| SES sa-east-1 | identical US$0.10/1k in all regions |
| Health endpoint | none in codebase — monitor homepage + event page |
| Hostinger SP datacenter | "América do Sul" advertised — **confirm at checkout** |

## Sources
- Hostinger VPS: https://www.hostinger.com/br/vps (KVM2 R$42.99, 17/jul/2026)
- Locaweb VPS: https://www.locaweb.com.br/servidor-vps/ (4GB R$53.90 promo)
- KingHost VPS: https://king.host/servidor-vps (4GB R$89.90)
- Lightsail: https://aws.amazon.com/lightsail/pricing/
- USD/BRL spot: https://br.investing.com/currencies/usd-brl (R$5.10)
- IOF 3.5%: https://www.cartoesdecredito.me/cartoes/como-ficou-o-iof-dos-cartoes-de-credito-em-2026/ · https://agenciabrasil.ebc.com.br/economia/noticia/2025-06/entenda-como-fica-o-iof-apos-derrubada-de-decreto
- SES: https://aws.amazon.com/pt/ses/pricing/ · uniform: https://costgoat.com/pricing/amazon-ses
- Stripe BR: https://stripe.com/pricing · conta: https://support.stripe.com/questions/brazil-specific-information-to-open-a-stripe-account · Pix: https://docs.stripe.com/payments/pix
- registro.br: https://registro.br/dominio/
- Oracle free tier: https://www.infoq.com/news/2026/07/oracle-cloud-free-tier-limits/
- WhatsApp add-on: https://www.messagecentral.com/blog/whatsapp-business-api-pricing-brazil
