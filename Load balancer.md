# AWS Load Balancer vs API Gateway — Detailed Guide

## 🧭 Overview

Both **API Gateway** and **Application Load Balancer (ALB)** can route traffic to **AWS Lambda** functions — but they serve very different purposes:

- **API Gateway** → Focuses on API management and developer-facing features.
- **ALB** → Focuses on traffic routing and load balancing across services.

---

## 🏗️ High-Level Architecture

### API Gateway Flow

```text
Client → API Gateway → Lambda
```

### ALB Flow

```text
Client → Application Load Balancer → Target Group (Lambda/EC2/ECS)
```

---

## ⚙️ Key Differences

| Feature | **API Gateway** | **Application Load Balancer (ALB)** |
|----------|-----------------|------------------------------------|
| **Primary Purpose** | API management and lifecycle | Traffic routing and load balancing |
| **Integration Types** | Lambda, HTTP endpoints, VPC links | EC2, ECS, Lambda, IP targets |
| **Routing Basis** | Path, method, stage | Path, host |
| **Auth** | JWT, Cognito, IAM | OIDC (limited), no native JWT |
| **Rate Limiting & Quotas** | ✅ Built-in | ❌ Not available |
| **Request/Response Transformations** | ✅ Yes | ❌ No |
| **WebSockets Support** | ✅ Yes | ❌ No |
| **Caching** | ✅ Built-in | ❌ No |
| **Cost Model** | Per request | Per hour + LCU |
| **Best For** | Public APIs with auth, throttling | Internal/external web traffic, microservices |
| **Supports Multiple Backends** | ❌ (mostly Lambda/HTTP) | ✅ (Lambda, ECS, EC2 mixed) |

---

## 🧩 How Terraform Configuration Works (ALB → Lambda)

```hcl
resource "aws_lambda_function" "orders" {
  function_name = "orders-fn"
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  filename      = "orders.zip"
  role          = aws_iam_role.lambda_exec.arn
}

resource "aws_lb_target_group" "orders_tg" {
  name        = "tg-orders"
  target_type = "lambda"
}

resource "aws_lb_target_group_attachment" "orders_attach" {
  target_group_arn = aws_lb_target_group.orders_tg.arn
  target_id        = aws_lambda_function.orders.arn
}

resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.app.arn
}

resource "aws_lb_listener_rule" "orders_rule" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.orders_tg.arn
  }

  condition { path_pattern { values = ["/orders*"] } }
}
```

## 🧠 How It Works

1. Client makes an HTTPS request.
2. ALB Listener (port 443) receives the request.
3. Listener Rule checks path (e.g., `/orders`).
4. Target Group (type = Lambda) forwards to the function.
5. Lambda executes and returns the response.

---

## 🌐 Combined Architecture: CloudFront + API Gateway + ALB + Lambda

```text
                ┌─────────────────────┐
   Client ▶▶▶▶▶ │   CloudFront (CDN)  │
                │   + WAF (Security)  │
                └─────────┬───────────┘
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
┌───────▼────────┐                  ┌───────▼───────────┐
│ API Gateway     │                  │ Application LB    │
│ (API mgmt layer)│                  │ (traffic router)  │
│                 │                  │                   │
│  • Auth (JWT)   │                  │  • Path-based     │
│  • Rate limits  │                  │  • Host-based     │
│  • Transforms   │                  │  • Mix backends   │
└───────┬────────┘                  └────────┬──────────┘
        │                                    │
        │                                    │
   ┌────▼─────┐                          ┌────▼──────┐
   │ Lambda   │                          │ Lambda   │
   │ orders   │                          │ products │
   └──────────┘                          └──────────┘
                                           (or ECS/EC2 too)
```

---

## 🧭 Choosing Between API Gateway and ALB

| Use Case | Recommended Service |
|----------|-------------------|
| Need authentication, rate limiting, quotas | API Gateway |
| Need to mix Lambda + ECS + EC2 backends | ALB |
| Want caching + WAF at edge | CloudFront in front of either |
| Need API keys, Swagger, or Dev Portal | API Gateway |
| Need host/path routing and load balancing | ALB |

---

## 🎯 Summary Analogy

| Service | Analogy |
|---------|---------|
| API Gateway | A toll booth and inspector — checks credentials, enforces policies, transforms requests |
| ALB | A traffic cop — routes cars (requests) to the right road (service) and balances load |

---

## ✅ Key Takeaways

- **API Gateway** routes API requests to Lambda or HTTP backends with full API lifecycle features.
- **ALB** routes and balances HTTP(S) traffic across Lambda, ECS, or EC2 targets.
- Both can coexist:
  - **CloudFront → API Gateway → Lambda** for public APIs.
  - **CloudFront → ALB → Lambda/ECS/EC2** for web and microservice routing.
- **Neither is a subset of the other** — they serve distinct but complementary roles.