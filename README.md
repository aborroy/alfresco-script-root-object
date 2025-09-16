# Alfresco Script Root Objects – Unified Addon (ACS 25.x)

A single **repository JAR** that exposes multiple **JavaScript Root Objects** for Alfresco Content Services (ACS). These objects let safe JavaScript (rules, repo web scripts, workflows) perform tasks that are blocked in sandboxed environments (e.g., `Data Dictionary > Scripts`) without using `Packages.*`.

## What’s inside (Root Objects)

* **`sysAdmin`** – Safe access to selected methods from `SysAdminParams`. 
* **`globalProperties`** – Read values from `alfresco-global.properties` and print all properties. 
* **`packagesScript`** – Minimal “escape hatch” to obtain WebApplicationContext and reflect class methods in a controlled way (`getContext()`, `getPackage()`, `getMethods()`). Use sparingly. 
* **`base64`** – Streamed Base64 encode/decode helpers for content/bytes, safe in rules. 
* **`renditionService2`** – Adds the missing `renditionService2` root object for modern renditions. 
* **`hylandProcess`** – Start **Hyland Automate** processes via OAuth2-secured REST. 

> Opinionated advice: reach first for **`sysAdmin`**, **`globalProperties`**, **`base64`**, and **`renditionService2`** for common tasks. Keep **`packagesScript`** as a last resort to inspect or bridge gaps. Use **`hylandProcess`** when integrating ACS with Hyland Automate.

## Compatibility

* **ACS**: 25.2.x (works for nearby 25.x with recompile if needed) 
* **Java**: 17 
* **Maven**: 3.8+ 

## Build

```bash
mvn clean package
# outputs: target/<artifact>.jar
```

## Deploy (recommended paths)

### Docker overlay (recommended)

Create a slim image layering the built JAR into the repository webapp:

```dockerfile
FROM alfresco/alfresco-content-repository-community:25.2.0
COPY target/*.jar /usr/local/tomcat/webapps/alfresco/WEB-INF/lib/
```

Then reference this image in your Compose stack. 

### Classic Tomcat/WAR

Copy the JAR into `WEB-INF/lib` and restart:

```bash
cp target/*.jar $TOMCAT_DIR/webapps/alfresco/WEB-INF/lib/
```

## Configuration (only needed for `hylandProcess`)

Add to `alfresco-global.properties`:

```properties
# OAuth token endpoint
hyland.oauth.url=https://auth.iam.experience.hyland.com/idp/connect/token

# OAuth client credentials
hyland.oauth.clientId=<client-id>
hyland.oauth.secret=<client-secret>

# Hyland process API endpoint (default app)
hyland.api.url=https://studio.experience.hyland.com/<process-app>/rb/v1/process-instances

# Default process key
hyland.default.processKey=Process_Default
```

You can override the API URL per call (see examples). 

## Usage Examples (Rhino JS)

### 1) `sysAdmin`

```javascript
logger.log(sysAdmin.getAlfrescoHost());
```

Use this instead of unsafe `Packages` access to `SysAdminParams`. 

### 2) `globalProperties`

```javascript
var host = globalProperties.get("alfresco.host");
logger.log(host);

// Dump everything to the JavaScript Console (use print)
print(globalProperties.all);
```

### 3) `packagesScript` (advanced / inspect only)

```javascript
var ctx = packagesScript.getContext();
var klass = packagesScript.getPackage("org.alfresco.repo.admin.SysAdminParams");
print(packagesScript.getMethods('org.alfresco.repo.admin.SysAdminParams'));
```

If you previously used:

```javascript
// Unsafe in ACS 25.x sandbox
var context = Packages.org.springframework.web.context.ContextLoader.getCurrentWebApplicationContext();
```

...replace with the `packagesScript` helpers above. 

### 4) `base64`

```javascript
// Stream-encode primary content
var b64 = base64.encode(document);
logger.log("B64 len: " + b64.length);

// Encode existing bytes (e.g., document.content)
var s = base64.encodeBytes(document.content);

// Decode back to Java byte[]
var bytes = base64.decodeToBytes(b64);
```

Tip: prefer `encode(document)` for large files—it streams and saves memory. 

### 5) `renditionService2`

```javascript
// List renditions for a node
var rs = renditionService2.getRenditions(document);
rs.forEach(function(r) { logger.log(r); });

// Get a rendition by name
logger.log(renditionService2.getRenditionByName(document, "pdf"));

// Fire-and-forget render
renditionService2.render(document, "pdf");
```

(Provides the modern `renditionService2` root object that’s otherwise missing.) 

### 6) `hylandProcess`

```javascript
// Using default app + default process key from properties
var vars = {
  invoiceNumber: document.properties["sap:invoiceNo"],
  amount: parseFloat(document.properties["sap:amount"] || 0),
  customFlag: true
};
logger.log(hylandProcess.startProcess("Process_1748835392417", vars));

// Override API URL for a different process app
var apiUrl = "https://studio.experience.hyland.com/<another-app>/rb/v1/process-instances";
logger.log(hylandProcess.startProcess(apiUrl, "Process_ABC123", vars));
```

Returns `true` on successful trigger (HTTP 2xx). Uses OAuth2 client credentials with token caching. 