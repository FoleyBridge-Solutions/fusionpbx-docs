# Messages

The Messages application enables FusionPBX to send and receive SMS and MMS messages through various providers. It provides a unified messaging interface that integrates with your PBX system, allowing users to manage text messaging alongside voice communications.

## Overview

The Messages application utilizes a message queue service to handle message delivery and reception. Key features include:

- **SMS and MMS Support**: Send and receive both text messages and multimedia messages
- **Multi-Provider Support**: Configure multiple SMS/MMS providers simultaneously 
- **Endpoint Integration**: Relay messages to registered endpoints including softphones and desk phones that support message queuing
- **Web Interface**: Send and manage messages through the FusionPBX web interface
- **User Assignment**: Route incoming messages to specific users or groups based on destination numbers

## Prerequisites

Before installing the Messages application, ensure:
- FusionPBX is installed and running
- You have superadmin access
- Your system has internet access to clone repositories
- You have an SMS/MMS provider account with API credentials

## Installation

### Step 1: Install Application Components

Clone the required repositories into your FusionPBX apps directory:

```bash
cd /var/www/fusionpbx/app
git clone https://github.com/fusionpbx/fusionpbx-app-messages.git messages
git clone https://github.com/fusionpbx/fusionpbx-app-providers.git providers
```

Run the upgrade script to register the new applications:

```bash
php /var/www/fusionpbx/core/upgrade/upgrade.php
```

### Step 2: Configure Application Defaults

Navigate to **Advanced** > **Upgrade** in the FusionPBX web interface and execute the following in order:

1. **App Defaults** - Installs default application settings
2. **Schema** - Creates necessary database tables
3. **Group Permissions** - Sets up access permissions
4. **Menu Defaults** - Adds menu items to the interface

### Step 3: Menu Configuration (Optional)

If the menu items were not automatically added, manually create them:

**Providers Menu Item:**
- Title: `Providers`
- Link: `/app/providers/providers.php`
- Parent Menu: `Accounts`
- Groups: `superadmin`

**Messages Menu Item:**
- Title: `Messages`
- Link: `/app/messages/messages.php`
- Parent Menu: `Applications`
- Groups: `superadmin`

### Step 4: Install System Services

The Messages application requires two system services to handle message processing:

#### Message Queue Service

Install and start the message queue service:

```bash
# Copy service file
cp /var/www/fusionpbx/app/messages/resources/service/debian-message_queue.service /etc/systemd/system/message_queue.service

# Enable and start the service
systemctl enable message_queue
systemctl start message_queue
systemctl daemon-reload
```

#### Message Events Service

Install and start the message events service:

```bash
# Copy service file
cp /var/www/fusionpbx/app/messages/resources/service/debian-message_events.service /etc/systemd/system/message_events.service

# Enable and start the service
systemctl enable message_events
systemctl start message_events
systemctl daemon-reload
```

Verify both services are running:

```bash
systemctl status message_queue
systemctl status message_events
```

### Step 5: Configure NGINX for MMS Support

To support MMS (multimedia messages), add a rewrite rule to your NGINX configuration:

1. Edit the NGINX configuration file:

```bash
nano /etc/nginx/sites-enabled/fusionpbx
```

2. Locate the `server` block for port 443 (HTTPS) and add the following rewrite rule. Place it after the REST API rewrite rule or before the phone vendor rewrite rules:

```nginx
server {
    listen 443;
    
    # ... other configuration ...
    
    # Message media rewrite for MMS support
    rewrite "^/app/messages/media/(.*)/(.*)" /app/messages/message_media.php?id=$1&action=download last;
    
    # ... other rewrite rules ...
}
```

3. Test the configuration and restart NGINX:

```bash
# Test configuration
nginx -t

# Restart NGINX if test passes
service nginx restart
```

## Configuration

### Default Settings Configuration

The Messages application uses Default Settings to configure how messages are processed. To access these settings:

1. Navigate to **Advanced** > **Default Settings**
2. Look for Category: **Messages**
3. Configure settings for both **inbound** and **outbound** subcategories

#### Inbound Message Settings

These settings control how incoming messages from your SMS provider are processed. Based on the screenshot, the following settings are available:

| Setting | Subcategory | Type | Name | Value | Enabled |
|---------|------------|------|------|--------|---------|
| Messages | inbound | content | message_content | Body | ✓ (sms) |
| Messages | inbound | content | message_from | From | ✓ (sms,mms) |
| Messages | inbound | content | message_media_array | | ✗ (mms) |
| Messages | inbound | content | message_media_name | | ✗ (mms) |
| Messages | inbound | content | message_media_type | | ✗ (mms) |
| Messages | inbound | content | message_media_url | | ✗ (mms) |
| Messages | inbound | content | message_to | To | ✓ (sms,mms) |
| Messages | inbound | format | message_from | +1xxxxxxxxxx | ✓ (sms,mms) |
| Messages | inbound | format | message_to | +1xxxxxxxxxx | ✓ (sms,mms) |

**Key Settings:**
- **message_content**: Maps the provider's message body field (typically "Body")
- **message_from**: Maps the sender's phone number field (typically "From")
- **message_to**: Maps the recipient's phone number field (typically "To")
- **message_media_*** settings: Used for MMS support (disabled by default)

#### Outbound Message Settings

These settings control how FusionPBX sends messages to your SMS provider:

| Setting | Subcategory | Type | Name | Value | Enabled |
|---------|------------|------|------|--------|---------|
| Messages | outbound | authentication | http_auth_password | | ✓ (auth) |
| Messages | outbound | authentication | http_auth_type | basic | ✓ (auth) |
| Messages | outbound | authentication | http_auth_username | | ✓ (auth) |
| Messages | outbound | content | content_type | post | ✓ |
| Messages | outbound | content | message_content | Body | ✓ (sms) |
| Messages | outbound | content | message_from | From | ✓ (sms,mms) |
| Messages | outbound | content | message_media_other | | ✗ (mms) |
| Messages | outbound | content | message_media_url | | ✗ |
| Messages | outbound | content | message_other | | ✗ (sms,mms) |
| Messages | outbound | content | message_to | To | ✓ (sms,mms) |
| Messages | outbound | destination | http_destination | https://api.twilio.com/2010-0... | ✓ |
| Messages | outbound | format | message_from | +1xxxxxxxxxx | ✓ (sms,mms) |
| Messages | outbound | format | message_to | +xxxxxxxxxxx | ✓ |
| Messages | outbound | format | message_to | +1xxxxxxxxxx | ✓ (sms,mms) |
| Messages | outbound | header | http_content_type | | ✗ |
| Messages | outbound | method | http_method | POST | ✓ |

**Key Settings:**
- **http_auth_username**: Your provider's username or Account SID
- **http_auth_password**: Your provider's password or Auth Token  
- **http_auth_type**: Set to `basic` for basic authentication
- **content_type**: Set to `post` for form-encoded data
- **http_destination**: The full API endpoint URL for your provider
- **http_method**: Usually `POST`

### Provider Setup

1. **Add a Provider**
   - Navigate to **Accounts** > **Providers**
   - Click the **ADD** button
   - Select your SMS/MMS provider from the list
   - Click **SETUP** to configure the provider

2. **Configure Provider Settings**
   - Enter your API credentials
   - Save the configuration

3. **Verify Default Settings**
   - Go to **Advanced** > **Default Settings**
   - Look for Category: `Messages`
   - Verify settings match the table above
   - Enable required settings (blue toggle = enabled)
   - Fill in authentication credentials:
     - `http_auth_username`: Your provider username/Account SID
     - `http_auth_password`: Your provider password/Auth Token
   - Verify `http_destination` contains your provider's API URL
   - Save all changes

### Destination Configuration

1. **Inbound Routing**
   - Navigate to **Dialplan** > **Destinations**
   - Assign SMS-enabled numbers to users or groups
   - Select the appropriate provider from the dropdown
   - Ensure the destination type is set correctly

2. **Country Code Settings**
   - Configure the default country code for your region
   - This enables matching with various number formats:
     - Local format (without country code)
     - National format (with country code)
     - E.164 format (international)

### User Configuration

1. **Extension Assignment**
   - Go to **Accounts** > **Extensions**
   - Ensure each user who needs messaging is assigned an extension
   - Enable messaging capabilities for the extension

2. **Permissions**
   - Verify user groups have appropriate messaging permissions
   - Check that users can access the Messages application

## Usage

### Sending Messages

1. **Web Interface**
   - Navigate to **Applications** > **Messages**
   - Click **New Message**
   - Enter recipient number(s)
   - Type your message (160 characters for SMS)
   - Attach media files for MMS (if supported by provider)
   - Click **Send**

2. **Compatible Endpoints**
   - Softphones and desk phones that support SIP MESSAGE
   - Devices must be registered and support the message queue protocol

### Receiving Messages

1. **Incoming Message Flow**
   - Messages arrive at your provider
   - Provider sends webhook/API call to FusionPBX
   - System matches destination to user/group
   - Message appears in web interface
   - Message relays to registered endpoints (if supported)

2. **Notifications**
   - Check web interface for new message indicators
   - Configure email notifications (if available)
   - Endpoint devices receive push notifications (device-dependent)

## Troubleshooting

### Common Issues

1. **Messages Not Sending**
   - Check Default Settings under **Advanced** > **Default Settings** > Category: **Messages**
   - Verify `http_auth_username` and `http_auth_password` are filled in
   - Ensure `http_destination` contains the correct API URL
   - Verify `http_method` is set to POST
   - Check service status: `systemctl status message_queue`
   - Confirm provider account has sufficient balance

2. **Messages Not Receiving**
   - Verify webhook URL is accessible from internet
   - Check NGINX rewrite rules are in place (see Installation section)
   - Confirm destination numbers are properly configured
   - Review provider webhook configuration

3. **MMS Not Working**
   - Enable MMS-related settings in Default Settings (message_media_array, message_media_url, etc.)
   - Ensure NGINX rewrite rule is configured
   - Verify provider supports MMS

4. **Authentication Errors**
   - In Default Settings, verify:
     - `http_auth_type` is set to `basic`
     - `http_auth_username` has correct value
     - `http_auth_password` has correct value
   - Check that settings are enabled (blue toggle)

### Service Management

```bash
# Check service status
systemctl status message_queue
systemctl status message_events

# View logs
journalctl -u message_queue -f
journalctl -u message_events -f

# Restart services
systemctl restart message_queue
systemctl restart message_events
```

## Security Considerations

- **API Credentials**: Store provider API keys securely, never commit to version control
- **IP Authentication**: Configure provider IP whitelisting where available
- **HTTPS Only**: Ensure all webhook endpoints use HTTPS
- **Rate Limiting**: Implement rate limiting to prevent abuse
- **User Permissions**: Restrict messaging access to authorized users only

## Provider Examples

### Twilio Configuration Example

Twilio is one of the most popular SMS/MMS providers. Here's a step-by-step guide to configure Twilio with FusionPBX:

#### Prerequisites for Twilio

1. **Twilio Account**
   - Sign up at [twilio.com](https://www.twilio.com)
   - Upgrade from trial account for production use
   - Note your Account SID and Auth Token from the Twilio Console

2. **Phone Number**
   - Purchase a phone number with SMS/MMS capabilities
   - Note the phone number for configuration

#### Twilio Setup Steps

1. **Add Twilio as Provider**
   - Navigate to **Accounts** > **Providers**
   - Click **ADD** and select **Twilio** from the list
   - Click **SETUP**

2. **Configure Twilio Credentials**
   
   Enter the following in the provider settings:
   
   ```
   Account SID: ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Auth Token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Phone Number: +1234567890 (your Twilio number)
   ```

3. **Configure Webhook URLs in Twilio**
   
   Log into your Twilio Console and set up webhooks:
   
   - Go to **Phone Numbers** > **Manage** > **Active Numbers**
   - Click on your phone number
   - In the **Messaging** section, configure:
   
   ```
   Webhook URL (when a message comes in):
   https://your-fusionpbx-domain.com/app/messages/resources/providers/twilio.php
   
   HTTP Method: POST
   
   Status Callback URL (optional):
   https://your-fusionpbx-domain.com/app/messages/resources/providers/twilio_status.php
   ```

4. **Configure FusionPBX Settings**
   
   In **Advanced** > **Default Settings**, add or verify:
   
   ```
   Category: messages
   Subcategory: providers
   Type: array
   Name: twilio
   Value: {"account_sid":"ACxxxx","auth_token":"xxxx","phone_number":"+1234567890"}
   ```

5. **Assign Number to Destination**
   - Go to **Dialplan** > **Destinations**
   - Create or edit a destination with your Twilio number
   - Set **Type**: `Inbound Route`
   - Set **Number**: Your Twilio phone number (without +)
   - Assign to a user or group
   - Select **Twilio** as the provider

6. **Test the Configuration**
   
   **Test Receiving:**
   - Send an SMS from your mobile phone to the Twilio number
   - Check **Applications** > **Messages** for the received message
   
   **Test Sending:**
   - Go to **Applications** > **Messages**
   - Click **New Message**
   - Enter a recipient mobile number
   - Type a test message
   - Click **Send**

#### Twilio Troubleshooting

**Common Issues:**

1. **401 Authentication Error**
   - Verify Account SID and Auth Token are correct
   - Ensure credentials have no extra spaces
   - Check that account is active and funded

2. **Messages Not Receiving**
   - Verify webhook URL is publicly accessible
   - Check Twilio Console for webhook errors
   - Ensure SSL certificate is valid
   - Review Twilio debugger at console.twilio.com

3. **Messages Not Sending**
   - Confirm phone number is SMS-capable
   - Verify recipient number format (+1234567890)
   - Check Twilio account balance
   - Review geographic permissions in Twilio Console

**Viewing Twilio Logs:**

In Twilio Console:
- Navigate to **Monitor** > **Logs** > **Messaging**
- Check for errors or failed attempts
- Use the Twilio Debugger for detailed error messages

**Testing Webhook Connectivity:**

```bash
# Test if your webhook is accessible
curl -X POST https://your-fusionpbx-domain.com/app/messages/resources/providers/twilio.php \
  -d "From=+1234567890" \
  -d "To=+0987654321" \
  -d "Body=Test message"
```

### Plivo Configuration Example

#### Prerequisites for Plivo

1. **Plivo Account**
   - Sign up at [plivo.com](https://www.plivo.com)
   - Note your Auth ID and Auth Token from the console
   - Add credits to your account

2. **Phone Number**
   - Purchase a Plivo phone number with SMS/MMS capabilities
   - Enable SMS for the number in Plivo console

#### Plivo Setup Steps

1. **Add Plivo as Provider**
   - Navigate to **Accounts** > **Providers**
   - Click **ADD** and select **Plivo**
   - Click **SETUP**

2. **Configure Plivo Credentials**
   ```
   Auth ID: MAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Auth Token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Phone Number: +1234567890 (your Plivo number)
   ```

3. **Configure Plivo Application**
   
   In Plivo Console:
   - Go to **Messaging** > **Applications**
   - Create a new application
   - Set Message URL:
   ```
   https://your-fusionpbx-domain.com/app/messages/resources/providers/plivo.php
   Message Method: POST
   ```
   - Link your phone number to this application

4. **Assign Number in FusionPBX**
   - Go to **Dialplan** > **Destinations**
   - Create destination with your Plivo number
   - Select **Plivo** as the provider
   - Assign to user/group

5. **Test Configuration**
   ```bash
   # Test webhook
   curl -X POST https://your-fusionpbx-domain.com/app/messages/resources/providers/plivo.php \
     -d "From=1234567890" \
     -d "To=0987654321" \
     -d "Text=Test message"
   ```

### Telnyx Configuration Example

#### Prerequisites for Telnyx

1. **Telnyx Account**
   - Sign up at [telnyx.com](https://www.telnyx.com)
   - Create an API key in Mission Control Portal
   - Note your API key and public key

2. **Messaging Profile**
   - Create a messaging profile
   - Purchase a phone number with SMS/MMS
   - Assign number to messaging profile

#### Telnyx Setup Steps

1. **Add Telnyx as Provider**
   - Navigate to **Accounts** > **Providers**
   - Click **ADD** and select **Telnyx**
   - Click **SETUP**

2. **Configure Telnyx Credentials**
   ```
   API Key: KEYxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Public Key: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   Phone Number: +1234567890
   Messaging Profile ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   ```

3. **Configure Webhook in Telnyx**
   
   In Telnyx Mission Control:
   - Go to **Messaging** > **Messaging Profiles**
   - Edit your profile
   - Set Webhook URL:
   ```
   Inbound Webhook URL:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/telnyx.php
   
   Webhook API Version: v2
   Webhook Failover URL: (optional backup URL)
   ```

4. **Configure Outbound Settings**
   - Enable delivery receipts
   - Set webhook for status callbacks
   - Configure retry attempts

5. **Test with Telnyx CLI**
   ```bash
   # Install Telnyx CLI
   npm install -g @telnyx/cli
   
   # Test sending
   telnyx messages create \
     --from="+1234567890" \
     --to="+0987654321" \
     --text="Test message"
   ```

### Vonage (Nexmo) Configuration Example

#### Prerequisites for Vonage

1. **Vonage Account**
   - Sign up at [vonage.com](https://www.vonage.com)
   - Get API Key and API Secret from dashboard
   - Generate application with private key

2. **Virtual Number**
   - Rent a virtual number with SMS capability
   - Link number to your application

#### Vonage Setup Steps

1. **Add Vonage as Provider**
   - Navigate to **Accounts** > **Providers**
   - Click **ADD** and select **Vonage** or **Nexmo**
   - Click **SETUP**

2. **Configure Vonage Credentials**
   ```
   API Key: xxxxxxxx
   API Secret: xxxxxxxxxxxxxxxx
   Phone Number: +1234567890
   Application ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   Private Key: (paste contents of private.key file)
   ```

3. **Configure Webhooks in Vonage**
   
   In Vonage Dashboard:
   - Go to **Your Applications**
   - Edit your application
   - Set webhooks:
   ```
   Inbound URL:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/vonage.php
   
   Status URL:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/vonage_status.php
   
   HTTP Method: POST
   ```

4. **Link Number to Application**
   - Go to **Numbers** > **Your Numbers**
   - Click settings for your number
   - Link to your application

5. **Test with Vonage CLI**
   ```bash
   # Install Vonage CLI
   npm install -g @vonage/cli
   
   # Configure CLI
   vonage config:set --apiKey=xxx --apiSecret=xxx
   
   # Send test message
   vonage sms --from="1234567890" --to="0987654321" --message="Test"
   ```

### Bandwidth Configuration Example

#### Prerequisites for Bandwidth

1. **Bandwidth Account**
   - Sign up at [bandwidth.com](https://www.bandwidth.com)
   - Get Account ID, Username, and Password
   - Create API credentials

2. **Application Setup**
   - Create messaging application
   - Order phone number through dashboard
   - Associate number with application

#### Bandwidth Setup Steps

1. **Add Bandwidth as Provider**
   - Navigate to **Accounts** > **Providers**
   - Click **ADD** and select **Bandwidth**
   - Click **SETUP**

2. **Configure Bandwidth Credentials**
   ```
   Account ID: xxxxxxx
   Username: your_username
   Password: your_password
   Application ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   Phone Number: +1234567890
   API Token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   API Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

3. **Configure Application in Bandwidth**
   
   In Bandwidth Dashboard:
   - Go to **Applications**
   - Edit your messaging application
   - Configure callbacks:
   ```
   Messaging Callback URL:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/bandwidth.php
   
   Status Callback URL:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/bandwidth_status.php
   ```

4. **Configure Phone Number**
   - Go to **Phone Numbers**
   - Select your number
   - Enable SMS and MMS features
   - Assign to your application

5. **Test Configuration**
   ```bash
   # Test with curl
   curl -X POST "https://messaging.bandwidth.com/api/v2/users/{accountId}/messages" \
     -H "Content-Type: application/json" \
     -u "username:password" \
     -d '{
       "applicationId": "your-app-id",
       "to": ["+10987654321"],
       "from": "+11234567890",
       "text": "Test message"
     }'
   ```

### SignalWire Configuration Example

#### Prerequisites for SignalWire

1. **SignalWire Account**
   - Sign up at [signalwire.com](https://www.signalwire.com)
   - Get Project ID, Auth Token, and Space URL
   - Note your Space name (subdomain)

2. **Phone Number**
   - Purchase a phone number with SMS/MMS
   - Configure number settings

#### SignalWire Setup Steps

1. **Add SignalWire as Provider**
   - Navigate to **Accounts** > **Providers**
   - Click **ADD** and select **SignalWire**
   - Click **SETUP**

2. **Configure SignalWire Credentials**
   ```
   Project ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   Auth Token: PTxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Space URL: yourspace.signalwire.com
   Phone Number: +1234567890
   ```

3. **Configure Webhooks in SignalWire**
   
   SignalWire uses LaML (compatible with TwiML):
   - Go to **Phone Numbers**
   - Click on your number
   - Set messaging webhook:
   ```
   When a Message Comes In:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/signalwire.php
   
   Method: POST
   
   Status Callback:
   https://your-fusionpbx-domain.com/app/messages/resources/providers/signalwire_status.php
   ```

4. **Test with SignalWire Client**
   ```bash
   # SignalWire is Twilio-compatible
   # Use Twilio SDK with SignalWire endpoint
   curl -X POST "https://yourspace.signalwire.com/api/laml/2010-04-01/Accounts/{ProjectID}/Messages.json" \
     -u "ProjectID:AuthToken" \
     -d "From=+1234567890" \
     -d "To=+0987654321" \
     -d "Body=Test message"
   ```

### Custom Provider Configuration

#### Setting Up a Custom Provider

For providers not listed above, you can configure a custom provider:

1. **Gather Provider Information**
   - API documentation
   - Authentication method (API key, OAuth, Basic Auth)
   - API endpoints for sending/receiving
   - Webhook requirements
   - Rate limits and restrictions

2. **Create Provider Template**
   - Navigate to **Accounts** > **Providers**
   - Click **Add Provider**
   - Configure basic settings:
   ```
   Provider Name: CustomSMS
   API URL: https://api.customprovider.com/v1/messages
   Auth Type: bearer_token / api_key / basic_auth
   Auth Credentials: your_credentials
   ```

3. **Configure Request Format**
   ```json
   {
     "send_message": {
       "method": "POST",
       "url": "https://api.provider.com/messages",
       "headers": {
         "Authorization": "Bearer {api_key}",
         "Content-Type": "application/json"
       },
       "body": {
         "from": "{from_number}",
         "to": "{to_number}",
         "message": "{message_text}"
       }
     }
   }
   ```

4. **Configure Webhook Handler**
   - Create PHP handler in `/app/messages/resources/providers/`
   - Parse incoming webhook data
   - Map to FusionPBX message format
   - Handle delivery receipts

5. **Test Custom Provider**
   ```php
   // Test script for custom provider
   <?php
   $provider = new CustomProvider($credentials);
   $result = $provider->send(
     '+1234567890',  // from
     '+0987654321',  // to
     'Test message'   // text
   );
   var_dump($result);
   ?>
   ```

## Provider Comparison

| Provider | SMS | MMS | Global | Pricing | API Quality | Support |
|----------|-----|-----|--------|---------|-------------|---------|
| Twilio | ✓ | ✓ | ✓ | $$ | Excellent | Excellent |
| Plivo | ✓ | ✓ | ✓ | $ | Very Good | Good |
| Telnyx | ✓ | ✓ | ✓ | $ | Excellent | Very Good |
| Vonage | ✓ | ✓ | ✓ | $$ | Very Good | Good |
| Bandwidth | ✓ | ✓ | US | $ | Good | Good |
| SignalWire | ✓ | ✓ | ✓ | $ | Excellent | Good |

## Troubleshooting All Providers

### Common Issues Across Providers

1. **SSL Certificate Issues**
   ```bash
   # Test SSL certificate
   openssl s_client -connect your-fusionpbx-domain.com:443
   ```

2. **Webhook Accessibility**
   ```bash
   # Test from external server
   curl -I https://your-fusionpbx-domain.com/app/messages/resources/providers/provider.php
   ```

3. **Provider API Status**
   - Check provider's status page
   - Monitor API response times
   - Review rate limit headers

4. **Debug Mode**
   
   Enable debug logging in FusionPBX:
   ```
   Advanced > Default Settings
   Category: messages
   Subcategory: debug
   Value: true
   ```
   
   Then check logs:
   ```bash
   tail -f /var/log/fusionpbx/messages_debug.log
   ```

For additional provider support and community configurations, visit the [FusionPBX Forums](https://www.fusionpbx.com/support/forum/) or [GitHub repository](https://github.com/fusionpbx).