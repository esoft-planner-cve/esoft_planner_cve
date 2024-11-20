# Vulnerability Dislosure: eSOFT Planner (3.24.08271-USA)

## Description

The company [eSOFT](https://www.esoftplanner.com/) makes a SAAS solution called Planner. The product is a facilities management tool aimed at sports facilities (swimming, baseball)
to help coordinate rentals, leagues, coaching sessions and more. There does not seem to be a downloadable or 'demo' version available to the public.
While evaluating eSOFT Planner, an access key is required. This access key is unique to each SAAS instance (each paying company gets an access key to place on their website
so that the correct esoftplanner instance is accessed). Version 3.24.08271-USA was tested.

### Google Dork

gdork: `https://www.esoftplanner.com/v3/planner/login.php?access=`

### Instance

While evaluating the product, an access key looks similar to `0A0aA0a0AaA0AaA0aA0AaA0Aaa==` was used. Using the following link will get you directly to the `user` experience for
the product: https://www.esoftplanner.com/v3/planner/login.php?access=0A0aA0a0AaA0AaA0aA0AaA0Aaa==

# Findings

## CVE-2024-48530: Authenticated Potential DoS in Instructor Appointment Availability

### Overview

An issue in the Instructor Appointment Availability module of eSoft Planner 3.24.08271-USA may allow attackers to
cause a Denial of Service (DoS) via a crafted POST request.

### Details

When viewing Instructor Appointment Availability, two dates are formed. When clicking search, a request body is formed for the POST request: `action=date&date10_month=01&date01_date=01&date10_year=2021&date11_month=01&date11_date=01&date11_year=2021`.
The page looks like the following:

![image](https://github.com/user-attachments/assets/64fb10c1-fdd3-4778-831e-131eeb193e1e)

Changing the `date10_month` value to `09'` (add a single quote similar to SQLi) causes the response to default to 1970 as the start date. The server then attempts to create an HTML table for every hour of every day since 1970.

Depending on the number of instructors and their appointment types, this can make for a very large response for the server to generate and send. An example one may be 250mb or larger, but during that time Cloudflare will either respond that the server isn't avaialble/responding, or no response will be received.

The following screenshot shows a CURL request with date being sent out to retrieve the large HTML Table, and a browser attempting to also reach the site, but getting timed out.

![image](https://github.com/user-attachments/assets/e7a676ec-006b-49fc-9d46-62ab39cc3684)

This screenshot shows burp browser requests for the site timing out, and finally responding during the window that CURL was retrieving the large HTML output.

![image](https://github.com/user-attachments/assets/cda78ae0-8440-44b6-ba24-9357a615a6ff)

It is UNKNOWN if this potential DoS is only for the user making the request or the server as a whole. Once this was discovered, it was not further exploited or explored due to potential loss of business and ethical concerns.

## CVE-2024-48531: Authenticated Reflected XSS in Duration field of rental_availability.php

### Overview

A reflected cross-site scripting (XSS) vulnerability on the Rental Availability module of eSoft Planner 3.24.08271-USA allows attackers to execute arbitrary code in the context of a user's browser via injecting
a crafted payload. 

### Details

When searching for rental space, the `duration` field is vulnerable to authenticated reflected XSS. The `loc_type` value must be present at the specific site for the xss to execute properly. An example URL of the XSS is: `https://www.esoftplanner.com/v3/planner/rental_availability.php?loc_type=111&duration=30%3Cscript%3Ealert(1)%3C/script%3E&date=2024-09-07`

![image](https://github.com/user-attachments/assets/fbf42ff0-68c1-45e8-9a26-4a6d282f0558)

## CVE-2024-48533: Unauthenticated User Email Enumeration

### Overview

A discrepancy between responses for valid and invalid e-mail accounts in the Forgot your Login? module of eSoft Planner 3.24.08271-USA allows attackers to enumerate valid user e-mail accounts.

### Details

When using the 'Forgot your Login?' feature, there is no captcha to prevent brute forcing. Additionally, if an email address is given which doesn't have a username, the message `Email address not found` is received. This looks as such:

![image](https://github.com/user-attachments/assets/725b6c82-6381-4b1b-ba2b-917d07c6c292)

However, one that is registered receives a positive message: `The email has been sent to`.

![image](https://github.com/user-attachments/assets/77d20a75-8b10-49b6-bf8e-233038e375ba)

While this HTML Form based request is abusable by itself, this can be simplified by using the ajax email checker functionality. You'll need to know the `client_key` for the site, which can be found in the source code of the main login page (and likely many others). The following screenshot shows where this can be found in the login page source code.

![image](https://github.com/user-attachments/assets/ed9b730a-2b7a-4429-95ee-26cea17a2531)

With this information, you can call the API w/o authentication to find a valid account:

![image](https://github.com/user-attachments/assets/17c2ab21-528a-450a-a8ad-0b32c27141c4)

vs an account which doesn't exist (empty HTML response)

![image](https://github.com/user-attachments/assets/d674d990-582b-407c-b118-d01b084a6315)

This method will also prevent a possible email trigger alerting a user.

## CVE-2024-48534: Authenticated Open Redirect & XSS in page_from field of campdetails.php

### Open Redirect

When viewing a camp details, a "Back" button appears, which is populated from the `page_from` variable within the URL. A normal request will populate with camplist as seen here:

![image](https://github.com/user-attachments/assets/d887ba6b-ff93-4fe9-ac10-2fb09bee2869)

However, we can break out of the `.php` tail to browse to `yahoo.com` with a payload of `https://www.yahoo.com"'><"`

While not pretty (some leftover garbage on the screen), it would be trivial to fix up this attack to be invisible to the user. The URL used was: `https://www.esoftplanner.com/v3/planner/campdetails.php?id=EXAMPLE&page_from=https://www.yahoo.com%22%27%3E%3C%22&page_month=0`. Clicking the Back button will bring the user to yahoo.com.

![image](https://github.com/user-attachments/assets/079bafc9-42f6-4300-b36e-249e60678226)

### XSS

A reflected cross-site scripting (XSS) vulnerability on the Camp Details module of eSoft Planner 3.24.08271-USA allows attackers to execute arbitrary code in
the context of a user's browser via injecting a crafted payload. This field is also suseptible to 

This field is also vulnerable to a reflected XSS. Using the following URL we're able to execute an example alert box: `https://www.esoftplanner.com/v3/planner/campdetails.php?id=EXAMPLE&page_from=%22%27/%3E%3Cscript%3Ealert(1)%3C/script%3E%3C%27%22&page_month=0`

![image](https://github.com/user-attachments/assets/629fa3d0-e589-435d-bac6-c8472b423f7f)

## CVE-2024-48535: Authenticated Stored XSS in Name fields

### Overview

A stored cross-site scripting (XSS) vulnerability in eSoft Planner 3.24.08271-USA allows attackers to execute arbitrary web scripts or HTML via injecting a crafted payload into the Name parameter.

### Details

The `First Name`, and `Last Name` fields are limited to 25 characters (even though the input box on the profile page is limited to `50`). Both fields can hold a minimal XSS payload for testing. Here is a screenshot of the `My Profile` page with XSS test strings loaded into it.

![image](https://github.com/user-attachments/assets/df71477b-0f90-43af-89c3-87718b4c8272)

The Save command sends a POST request with the fields:

![image](https://github.com/user-attachments/assets/7ff6516c-019a-44a5-b556-12d81ff3cd9a)

The injection is rendered at the top of the page where the Online Planner text is. The fields were changed to `first name` and `last name` to show the injection point in this next screenshot.

![image](https://github.com/user-attachments/assets/8293cd16-3709-402d-8dfd-84a2cd83e8b0)

Upon saving and reloading the page, the XSS examples execute.

![image](https://github.com/user-attachments/assets/731d323b-2367-415a-bb64-0ffd98116219)

![image](https://github.com/user-attachments/assets/66a903e2-aedf-4be2-ae3a-35f53fdb33b2)

These fields are likely rendered on admin pages, receipt pages, and potentially exploitable for privilege escalation.

This vulnerability is also likely exploitable when adding children's names on other pages. However, this was not explored.

## CVE-2024-48536: Authenticated Receipt Information Disclosure

### Overview

Incorrect access control in eSoft Planner 3.24.08271-USA allow attackers to view all transactions performed by the company via supplying a crafted web request.

### Details

When viewing a receipt, the following URL is used: `https://www.esoftplanner.com/v3/planner/view_receipt2.php?oid=<transaction_id>`

However, putting in a `transaction_id` which you aren't authorized to view (+1 or -1 of a valid id, or just `-1`) will load ALL transactions performed by that company. While most of the balances seem to be `$0.00`, some are filled in for an unknown reason.

Based on Firefox's Search functionality, we can see roughly 21,000 transactions have occured.

![image](https://github.com/user-attachments/assets/9e67c9d7-405c-4f4e-ae11-2a2e87700318)

If we change the URL to use `view_receipt.php` (not 2), which is used for Refundable Cancellation viewing, we also see the Notes/Special Instructions field. This field may contain customer PII leakage.

![image](https://github.com/user-attachments/assets/b4c8c397-d8ae-4b23-b383-6e18619bf1c7)

![image](https://github.com/user-attachments/assets/6854b4b3-e709-48df-9cd1-e635d3a13dc7)
