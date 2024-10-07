In OWASP ZAP, the **Full Scan** is a powerful feature that automates the process of identifying a wide range of vulnerabilities, including SQL Injection. While the full scan covers many types of attacks, you can **focus** your scan on specific parts of your application or parameters using **ZAP contexts**. This is especially useful when you want to target certain types of vulnerabilities, like SQL Injection, more precisely.

Here's a detailed explanation of how the **`--context`** option works and how it can help focus on SQL Injection:

### 1. **What is a ZAP Context?**
A **ZAP context** is essentially a set of rules or configurations that define **specific areas of an application** or web target that should be included or excluded during scanning. It allows you to:
- Define **authentication rules** (e.g., login forms).
- Specify **in-scope URLs** or restrict the scan to a particular part of the site.
- Focus on certain **parameters** or areas where SQL Injection or other vulnerabilities are more likely.
  
By using a **context**, you can fine-tune the scan to focus only on the parts of the site or API where you think SQL Injection attacks are most likely to succeed, such as form fields, URLs with query parameters, and input validation logic.

### 2. **Why Use `--context` for SQL Injection?**
Although ZAP's **full scan** already checks for SQL Injection, creating a **custom context** can:
- Limit the scan to only **certain pages** or **inputs** where SQL Injection is more likely (e.g., login pages, forms, and search boxes).
- Exclude parts of the application that are **not relevant** (e.g., static pages).
- Ensure the scanner is more **efficient and faster** by narrowing its scope.
  
This way, ZAP won’t waste time scanning static content and will instead focus more deeply on dynamic input points.

### 3. **How to Create a Custom Context for SQL Injection?**
You can manually create a **ZAP context** using ZAP's GUI or through scripting, and it will define:
- **In-scope URLs**: These are the pages or endpoints you want to scan (e.g., login pages, search results).
- **Parameter rules**: Define which parameters (GET, POST) are to be tested for vulnerabilities.
- **Authentication**: If the site requires login, you can include credentials and login pages as part of the context.

Here’s an example of how a custom **context** might work for SQL Injection:
1. **Scope**: Set the scope to cover URLs with dynamic query parameters like `/login`, `/search`, or `/product?id=`.
2. **Testable Parameters**: Instruct ZAP to focus on input fields like `username`, `password`, or query parameters (`id`, `search`, etc.)—fields that are most susceptible to SQL Injection.
3. **Authentication**: If necessary, include a login sequence so that the scan can access authenticated pages where SQL Injection attacks are more likely.

### 4. **How to Use `--context` in ZAP?**
Once you've created a context file (let's say it's called `sql_injection.context`), you can pass it to the ZAP scan using the `--context` argument. This ensures ZAP performs the scan **within the context’s boundaries**. The file might look like this:

```xml
<context>
    <urls>
        <url>https://xyz-test.com/login</url>
        <url>https://xyz-test.com/search</url>
        <url>https://xyz-test.com/product?id=*</url>
    </urls>
    <authentication>
        <loginUrl>https://xyz-test.com/login</loginUrl>
        <username>testuser</username>
        <password>password</password>
    </authentication>
    <testableParameters>
        <parameter name="username"/>
        <parameter name="password"/>
        <parameter name="id"/>
        <parameter name="search"/>
    </testableParameters>
</context>
```

### 5. **Using the Context in Jenkins Pipeline**
In the Jenkins pipeline, when you run the **ZAP full scan**, you can specify the context as follows:

```sh
docker run -dt --name owasp-zap -p 8080:8080 -p 9090:9090 -p 80:8888 ghcr.io/zaproxy/zaproxy:stable

docker exec owasp-zap mkdir -p /zap/wrk

docker cp sql_injection.context owasp-zap:/zap/wrk

docker exec owasp-zap zap-full-scan.py -t https://dev.shipease.in  -r report.html /zap/wrk   ##for full scan 

docker exec owasp-zap zap-full-scan.py -t https://dev.shipease.in   -r sql_injection_report.html -I --context /zap/wrk/sql_injection.context ##for full scan SQL injections 
```

This command tells ZAP to:
- **Use the context file** (`sql_injection.context`) to guide its scanning.
- **Focus** the scan only on the areas (URLs, parameters) defined in the context.
- **Generate a report** specifically for SQL Injection (`sql_injection_report.html`).

### 6. **Practical Use Case**
Let’s say you have a website with both static and dynamic pages. You only want ZAP to test for SQL Injection on certain dynamic pages like:
- **Login pages** (`https://xyz-test.com/login`)
- **Product pages** with an `id` parameter (`https://xyz-test.com/product?id=1`)
- **Search functionality** (`https://xyz-test.com/search?query=test`)

By using a custom **context**, you tell ZAP to ignore pages like **About Us** or **Privacy Policy** and instead concentrate its effort on scanning these specific areas that are vulnerable to SQL Injection.

### 7. **Benefits of Using Contexts for SQL Injection**
- **Focus the scan** on high-risk areas of your application.
- **Reduce scan time** by excluding irrelevant parts of the site.
- **Improve accuracy** by focusing the scan on inputs/parameters known to be at risk.
- **Enable authentication** if the SQL injection vulnerability requires logging in to access protected pages.

### 8. **Summary of Steps:**
1. **Create a context**: Define which pages, parameters, and authentication flows ZAP should focus on.
2. **Reference the context** in your Jenkins pipeline with `--context`.
3. **Run the scan**: ZAP will now perform a full scan but limit itself to the scope you’ve defined, with a particular focus on SQL Injection.

This approach ensures that you’re conducting a more **targeted** and **efficient** SQL Injection test while still leveraging the full power of OWASP ZAP.

Let me know if you want help creating a specific context file or have further questions!
