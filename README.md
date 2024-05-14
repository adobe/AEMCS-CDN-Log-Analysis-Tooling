# AEMCS CDN Log Analysis Tooling

This repository helps to analyze the CDN logs of the AEM as a Cloud Service (AEMCS) environment using the [Splunk](https://www.splunk.com/en_us/products/observability-cloud.html) or [ELK stack](https://www.elastic.co/elastic-stack). The analysis provides insights into the performance and security of your AEMCS implementation.

## Dashboard Overview

To quickstart the analysis Adobe provides pre-built dashboards for both Splunk and ELK stack.

- **CDN Cache Hit Ratio**: provides insights into the total cache hit ratio and total count of requests by HIT, PASS, and MISS status. Also provides top HIT, PASS, and MISS URLs.

    ![CDN Cache Hit Ratio](images/CHR-dashboard.png)

- **CDN Traffic Dashboard**: provides insights into the traffic via CDN and Origin request rate, 4xx and 5xx error rates, and non-cached requests. Also provides max CND and Origin requests per second per client IP address and more insights to optimize the CDN configurations.

    ![CDN Traffic Dashboard](images/Traffic-dashboard.png)

- **WAF Dashboard**: provides insights via analyzed, flagged, and blocked requests. Also provides top attacks by WAF Flag ID, top 100 attackers by client IP, country, and user agent and more insights to optimize the WAF configurations.

    ![WAF Dashboard](images/WAF-Dashboard.png)

However, you can enhance and create additional dashboards to gain further insights and share them with the AEMCS community. Your contributions are welcome, please create a pull request and don't forget to review the [contributing guidelines](CONTRIBUTING.md).

## Getting Started

The following respective sections provide the necessary steps to get started with the analysis using the **ELK stack or Splunk**.

- [ELK stack](ELK/README.md)
- [Splunk](splunk/README.md)




