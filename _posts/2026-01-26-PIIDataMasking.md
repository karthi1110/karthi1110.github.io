---
title: How to Mask Sensitive Information Using Fluentd/Fluent Bit
description: Learn how to automatically mask PII, passwords, and sensitive data in logs using Fluentd and Fluent Bit. Step-by-step guide with working examples and Docker testing setup.
date: 2026-01-26 10:00:00 +0530
categories: [devops, monitoring, logging]
tags: [pii, data-masking, fluentd, fluent-bit]     # TAG names should always be lowercase
---

Imagine discovering that your application logs contain thousands of customer phone numbers, credit card details, and social security numbers—all in plaintext. This isn't a hypothetical scenario; it's a reality for many organizations that discover PII exposure only after a security audit or breach.

In this guide, you'll learn how to:
- Identify PII leakage in your logging pipeline
- Implement centralized data masking without code changes
- Configure Fluentd and Fluent Bit to automatically redact sensitive information
- Test your masking patterns before production deployment

The goal of this guide is convert structured logs that contain PII information like (mobile numbers, identity information, name etc…)
```
{"timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=1234512345 registered on aadhaarNumber=1234512345"}
```
to a format where this information is masked.
```
{"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=######2345 registered on aadhaarNumber=********"}
```

## Compliance and security risks
The problem persists primarily because developers add verbose logging for debugging purposes during development and these debug statements often go unnoticed in code reviews. Over time, as services evolve and multiple teams contribute code, sensitive data gradually accumulates in logs without anyone realizing the extent of PII exposure.

Unmasked sensitive data in logs creates significant compliance violations and security risks, as PII, passwords, and credit card numbers become accessible to anyone with log access, potentially leading to data breaches and regulatory fines. Organizations often discover this problem only after a security audit or incident, when they realize that their centralized logging systems have been storing plaintext sensitive information across multiple storage backends and retention periods.
 
To address this challenge **centrally without requiring individual service changes, we can leverage the existing Fluentd/Fluent** Bit that already ingests logs from all containers within the cluster. By implementing a filter plugin with **regex-based pattern matching** in the Fluentd/Fluent Bit pipeline, we can automatically mask PII data before logs are forwarded to the centralized logging system, eliminating the need for code changes across multiple services.

**Important Note:** Once data is masked at the Fluentd/Fluent Bit layer and stored in the destination logging system, it cannot be reverted back to its original unmasked form, so careful consideration of masking patterns is essential.

## Fluentd Implementation

### Selecting the Right Flunetd Plugin

- #### <a href="https://github.com/fluent/fluent-plugin-sanitizer" target="_blank">**fluent-plugin-sanitizer**</a>

Sanitizer is a custom plugin that must be installed and implemented to mask PII data by substituting the original value with its **hash through the application of regular expressions.**
Sanitizer is provided as RubyGem package and you can install sanitize plugin on both td-agent and OSS Fluentd easily.

```
### For td-agent user :
$ td-agent-gem install fluent-plugin-sanitizer

### For OSS Fluentd user :
$ fluent-gem install fluent-plugin-sanitizer
```
#### Configuring Fluentd with Sanitizer Plugin
Once the plugin is installed, you can configure Fluentd to use it in your configuration file (typically located at `/etc/fluent/fluent.conf` or `/etc/td-agent/td-agent.conf`).

```
<filter *>
      @type sanitizer
      <rule>
        keys message
        pattern_regex /^\d{3}-?\d{3}-?\d{4}$/
        pattern_regex_prefix phone
      </rule>
      <rule>
        keys message
        pattern_regex /\b(?:card\s+)?(\d{12,19})\b/
        pattern_regex_prefix card
      </rule>
</filter>
```
Sample logs before and after applying the sanitizer plugin:
```
sanitize 'Testing payment with card 5555666677778888' to 
'Testing payment with card card\_cfe02588c4fa9615b16d929aa2740343'

sanitize '123-123-1234' to 
'phone\_d90f42af4ea186d2f1e2c503358243e3'
   ```
- #### <a href="https://docs.fluentd.org/filter/record_transformer" target="_blank">**filter_record_transformer**</a>

This is an out-of-the-box filter plugin within fluentd that can be utilized to mask PII data through the application of regular expressions

**Pros:**
- Built-in to fluentd, no separate installation required.
- Highly flexible due to the use of regular expressions for pattern matching.
- **Partial data masking** is possible with regular expressions (regex). 

#### Configuring Fluentd with Record Transformer Plugin

Create a test configuration file `fluentd.conf`:

```
<source>
    @type dummy
    tag dummy.log
    dummy {"@timestamp":"2023-06-05T17:04:33.505+05:30","message":"Starting server on port 8080"}
</source>

<source>
    @type dummy
    tag dummy.log
    dummy {"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user","message":"Sending SMS to mobileNumber=1234512345 registered on aadhaarNumber=1234512345"}
</source>

<source>
    @type dummy
    tag dummy.log
    dummy {"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/bank","message":"Successfully registered mobileNumber=9876543210 to panNumber=ABCDE1234F"}
</source>

<filter **>
    @type record_transformer
    enable_ruby true
    <record>
      message ${record["message"].to_s.gsub(/mobileNumber=(\d{6})(\d{4})/, 'mobileNumber=######\2').gsub(/aadhaarNumber=[^,\s]+/, 'aadhaarNumber=********')}
    </record>
</filter>

<match **>
    @type stdout
</match>
```

Run the test using Docker:

```bash
docker run \
-v $(pwd)/fluentd.conf:/fluentd/etc/fluentd.conf \
fluent/fluentd:latest -c /fluentd/etc/fluentd.conf
```

Sample logs before and after applying the record_transformer plugin:
```
{"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=1234512345 registered on aadhaarNumber=1234512345"} ->

{"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=######2345 registered on aadhaarNumber=********"}
```

## Fluent Bit Implementation

### Selecting the Right Flunetd Plugin

- **Record Modifier:**

This plugin gives us the ability to modify our structured logs by replacing entire key, values with something else. But does not allow replacing a small part of the value. For example, take the below structured log 
```
{"timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=1234512345 registered on aadhaarNumber=1234512345"} 
```
in our case the message field of type string has mobileNumber in it. We just want to replace this value. But this filter can only replace the entire message content.

- **Lua:** 

This filter allows you to modify the incoming records using custom <a href="https://docs.fluentbit.io/manual/data-pipeline/filters/lua" target="_blank">Lua scripts</a>. This helps us to extend Fluent Bit capabilities by writing custom filters using Lua programming language. We can use this to write custom Lua script to perform search & replace operation for us.

#### Writing the Lua Script
Create a file called mask.lua in the same directory where fluent-bit.conf exists. Copy the below content inside mask.lua file.

```lua
function mask_sensitive_info(tag, timestamp, record)
      message = record["message"]
      if message then
          -- Match "aadhaarNumber:xxxx," and replace with "aadhaarNumber:****,"
          local masked_message = string.gsub(message, 'aadhaarNumber=[^,]*', 'aadhaarNumber=****')

          -- Match "mobileNumber:xxxx," and replace with "mobileNumber:######s,"
          masked_message = string.gsub(masked_message, 'mobileNumber=(%d%d%d%d%d%d)(%d%d%d%d)', 'mobileNumber=######%2')

          record["message"] = masked_message
      end
      return 2, timestamp, record
end
```
#### Using the Lua Script In Lua Plugin

To enable the lua plugin, you need to add it to the Fluent Bit configuration file. Below is an example of how to configure the lua pugin to mask the PII field in the log data.

```
[INPUT]
      Name   dummy
      dummy  {"@timestamp":"2023-06-05T17:04:33.505+05:30","message":"Staring server on port 8080"}
      Tag    dummy.log

[INPUT]
      Name   dummy
      dummy  {"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user","message":"Sending SMS to mobileNumber=1234512345, registered on aadhaarNumber=1234512345"}
      Tag    dummy.log

[INPUT]
      Name   dummy
      dummy  {"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/bank","message":"Successfully registered mobileNumber=1234512345, to panNumber=1234512345"}
      Tag    dummy.log

[FILTER]
      Name    lua
      Match   *
      call    mask_sensitive_info
      script  /fluent-bit/scripts/mask.lua

[OUTPUT]
      Name   stdout
      Match  *
```
Run the test using Docker:
```bash
docker run \
    -v $(pwd)/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf \
    -v $(pwd)/mask.lua:/fluent-bit/scripts/mask.lua \
    -ti fluent/fluent-bit \
    /fluent-bit/bin/fluent-bit \
    -c /fluent-bit/etc/fluent-bit.conf
```

Sample logs before and after applying the lua plugin:
```
{"timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=1234512345 registered on aadhaarNumber=1234512345"} ->

{"@timestamp":"2023-06-05T17:04:33.505+05:30","requestURI":"/api/user",
"message":"Sending SMS to mobileNumber=######2345 registered on aadhaarNumber=********"}
```

## Conclusion

By implementing centralized PII masking at the Fluentd/Fluent Bit layer, you can:
- Protect sensitive data across all services without code changes
- Achieve compliance with GDPR, HIPAA, and PCI-DSS requirements
- Reduce security audit findings
- Deploy masking patterns in hours instead of weeks

### Next Steps
- Audit your current logs for PII exposure
- Test masking patterns in a non-production environment
- Document your regex patterns for team review
- Set up monitoring to detect unmasked PII
- Schedule regular pattern updates as new fields are added

### Important Reminders
⚠️ **Test thoroughly**: Once masked, data cannot be recovered

⚠️ **Balance security and debugging**: Keep enough context for troubleshooting

⚠️ **Update patterns regularly**: New features may introduce new PII fields

## Related resources
- <a href="https://fluentd.ctc-america.com/blog/plugin-sanitizer" target="_blank">Fluentd Sanitizer Plugin Documentation</a>
- <a href="https://docs.fluentd.org/filter/record_transformer" target="_blank">Fluentd Record Transformer Plugin Documentation</a>
- <a href="https://docs.fluentbit.io/manual/data-pipeline/filters/lua" target="_blank">Fluent Bit Lua Filter Documentation</a>
- <a href="https://karthi1110.github.io/posts/GenerateFakeLogs/" target="_blank">Load testing Fluent Bit/Fluentd</a>

---
<h3 style="text-align:center;"> Subscribe to Level Up </h3>

<div style="text-align: center;">
<iframe src="https://karthikeyangopi.substack.com/embed" width="480" height="250" frameborder="0" scrolling="no"></iframe>
</div>