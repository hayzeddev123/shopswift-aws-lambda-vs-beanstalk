# ShopSwift: AWS Lambda vs Elastic Beanstalk

I evaluated AWS Lambda and Elastic Beanstalk as hosts for our small e-commerce site. I used a simple static storefront (HTML, CSS, and JavaScript) so I could compare the two platforms on the same workload.

I did not leave live AWS resources running. This report is the evaluation. Delete anything you create after you try the steps.

## What I understood

We are on a traditional web server today. I needed a cheaper, more scalable AWS option for a catalog-style site that will stay quiet most of the time and spike when we run a campaign.

The demo site is static. The cart lives in the browser. There is no checkout backend and no database.

## How I would deploy Lambda

I would package the site with a small Node.js handler and expose it with a Lambda Function URL.

- The handler returns each HTML, CSS, and JS file with the right content type.
- I would use ARM (Graviton), 128–256 MB memory, and a short timeout.
- SAM or CloudFormation would create the function and the HTTPS URL.

Lambda only runs when someone hits the site, so idle cost is near zero. Cold starts can make the first request after idle time a bit slower. Serving every image through Lambda is the wrong long-term design. In production I would put CloudFront in front, or host the files on S3 and keep Lambda for APIs.

**Delete after testing:** `sam delete`, then confirm the function, Function URL, and log group are gone.

## How I would deploy Elastic Beanstalk

I would wrap the same files in a tiny Express app on port 8080 and deploy it with the EB CLI (`eb init`, `eb create`).

Beanstalk itself has no platform fee. I pay for the EC2 instance, disk, public IPv4, and a load balancer if I add one. A single-instance environment is enough for this demo. A load-balanced environment costs more and is only worth it when I need high availability.

The server stays warm, so page time is steadier than Lambda. Scaling takes minutes, not milliseconds, and the instance bill keeps going at night.

**Delete after testing:** `eb terminate <env> --force`. Then check EC2, EBS, load balancers, and Elastic IPs. Forgotten Beanstalk environments are the usual surprise bill.

## Performance and scale

| Need | What I found |
|---|---|
| Quiet site, sudden spikes | Lambda scales out on its own |
| Steady traffic, consistent latency | Beanstalk stays warm |
| Global static speed | Neither. I would use CloudFront |
| Future login, checkout, long requests | Beanstalk (or containers), not Lambda as a web server |

A small Beanstalk `t3.micro` is fine for early traffic. Lambda is the better match if most hours of the month have almost no visitors.

## Cost

Lambda (us-east-1, published model): $0.20 per million requests after 1 million free, plus compute after 400,000 GB-seconds free each month. For a small catalog, I expect most months to land near $0–2.

Beanstalk: about $12/month for a single instance (instance + disk + public IPv4). With an Application Load Balancer I would budget closer to $29/month. That bill barely changes if traffic is low.

If I only need to serve HTML, CSS, and JS, S3 + CloudFront is cheaper than both, often a few dollars a month.

## What I recommend

For the current static site, I recommend Lambda (with CloudFront if we go public). I would not put a static catalog on Beanstalk just to keep a server running.

I would keep Beanstalk for the next phase, when we add server-side checkout or a real app process.

The architecture I want in 12 months:

- S3 + CloudFront for the storefront
- Lambda for APIs
- a database only when we have real orders
- Beanstalk or containers only if we outgrow functions

I would tag every resource, set budget alerts at $5 and $20, and tear the demo down the same day.
