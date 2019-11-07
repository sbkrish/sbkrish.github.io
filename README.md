# IPpay payment gateway integration with Agriios
![Imgur](https://i.imgur.com/2Nb2RT8.png)
### Prerequisites 
- Clone & configure Agriios Repository
- Install composer 
- Add Omnipay IPpay package

### What is IPpay?
IPpay is an important source solution for accepting Card-not-Present (CNP) payment transactions. Card not Present (CNP) refers to a purchase a consumer makes without
physically presenting his or her credit or debit card at the time of purchase.
### What is Agriios?
Agriios is a Point of Sale (POS) software for cannabis related products. An authorized POS user can create a purchese order, track order details and sell the product to consumer with the help of Agriios. 
### What is Composer?
Composer is a tool for dependency management in PHP. It allows you to declare the libraries your project depends on and it will manage (install/update) them for you. 
### What is Omnipay?
Omnipay is a payment processing library for PHP. It has been designed based on ideas from Active Merchant, plus experience implementing dozens of gateways for CI Merchant.
### Local setup
1. Clone Agriios repository from this [link][agCl] using HTTPS
	```
    $ git clone https://github.com/goorionlabs/agriios_1.0.git
    ```
2. Change the credentials in database.php and config.php files
3. Ensure the correct base_url in config.php
4.  [Install composer][composer]. 
	```
    $ sudo apt update && sudo apt install wget php-cli php-zip unzip curl
    $ curl -sS https://getcomposer.org/installer |php
    $ sudo mv composer.phar /usr/local/bin/composer
    $ composer
    ```
5. Add IPpay packages using composer.
	> This package is developed by the third party user on top of Omnipay libray

#### IPpay driver for the Omnipay PHP payment processing library
To install, simply add it to your composer.json file in application/third_party :  
```json
{
    "require": {
        "dranes/omnipay-ippay": "~1.1"
    }
}
```
And run composer to update your dependencies. For that follow the below steps:
1. Go to application/third_party from your terminal
2. And run the below commands
    ```sh
    $ curl -s http://getcomposer.org/installer | php
    $ php composer.phar update
    ```
3. All the existing packages were updated and newly added packages were installed. Ensure the below update commands for successfull installtion of IPpay.<br>
![Imgur](https://i.imgur.com/1YBrizn.png)
4. Check the newly added "dranes" folder inside application/third_party/vendor
    	

### Add new payment gateway in MySQL table
In the very beginning, we need to add IPpay in our database to enable it while making payment in the POS. 
##### geopos_gateways
There are 8 more payment gateways already added in this table. So we are adding IPpay in a 9th position.
```sql
INSERT INTO `geopos_gateways` (`id`, `name`, `enable`, `key1`, `key2`, `currency`, `dev_mode`, `ord`, `surcharge`, `extra`) VALUES ('9', 'IPpay', 'Yes', 'TestPay', 'TestKey', 'USD', 'true', '9', '0.00', 'none');
```
<u>Note:</u>
 - key1, key2 can be anything in string format
 - enable column must set 'true' 
 - ord is telling about the order which is going to be displayed in UI
 
### UI Design
![Imgur](https://i.imgur.com/Qeo6uTr.png)
    <center><u>Overview of IPpay gateway</u></center>

The above UI was designed based on the SecurePay payment gateway in Agriios.

1. Create a new file named as card_ippay.php under gateways in Views.
2. Copy and paste the source code from card_securepay.php
3. Add IPpay logo in png format and save as in 9.png inside assets/gateway_logo. 


### Make changes in Controller
After designing the IPpay gateway, we need to insert it in Billing controller.<br>
<u>Steps:</u>
1. Go to Billing.php in controllers
2. Find the function named as card()
3. Add ippay as case number 9 in switch case.
	```php
    case 9:
            $fname = 'ippay';
            break;
    ```

4. Create a new private function called as ippay() with required arguments. Set the Terminal id as 'TESTTERMINAL' and test mode as 'true'.
	```php
     private function ippay($cardNumber, $nmonth, $nyear, $cardCVC, $amount, $tid, $gateway_data) {
        $gateway = Omnipay::create('Ippay');
        $gateway->setTerminalId('TESTTERMINAL');
        $gateway->setTestMode(true);
        
        // Create a credit card object
        $card = new \Omnipay\Common\CreditCard(['number' => $cardNumber, 'expiryMonth' => $nmonth, 'expiryYear' => $nyear, 'cvv' => $cardCVC, ]);
        
        // Perform a purchase test
        $transaction = $gateway->purchase(['amount' => $amount, 'transaction_type' => 'SALE', 'card' => $card, ]);
        
        return $transaction->send();
    }
    ```

	> Note: The above code is for testing purpose only. The transaction has been done with test card data. The above code need to be changed while get into production.

5. To call the above function it should be added as 9th case inside the process_card() function and pass the transaction details as the parameters.
	```php
     case 9:
          $response = $this->ippay($cardNumber, $nmonth, $nyear, $cardCVC, $amount, $tid, $gateway_data);
          break;
    ```

### Changes in IPpay files
We need to make some changes in dranes/omnipay-ippay files to get everything working fine. Those files can be located in 					   
  ```
  application/third_party/vendor/dranes/omnipay-ippay/src/
  ```
##### 1. Purchase_request.php
Delete the source code inside getData() function as it was created for the transaction made by cheque. We need to change the data for card transaction. Copy and paste the below code inside that function.
```php
public function getData() {
     $this->getCard()->validate();
     $data = array();
     $data['TransactionType'] = $this->getTransactionType();
     $data['CardNum'] = $this->getCard()->getNumber();
     $data['CardExpMonth'] = $this->getCard()->getExpiryDate('m');
     $data['CardExpYear'] = $this->getCard()->getExpiryDate('y');
     $data['TotalAmount'] =floor($this->getAmount());
     return $data;
     }
```
##### 2. AbstractRequest.php
Endpoint is mentioned in this file to perform a RESTful API call.
```php
protected $sandboxEndpoint = "https://testgtwy.ippay.com/ippay";
protected $productionEndpoint = "https://gtwy.ippay.com/ippay";
```

POST data is constructed in XML format using dataSale() function. The test transaction POST data will look like this.
```xml
<ippay>
    <TransactionType>SALE</TransactionType>
    <TerminalID>TESTTERMINAL</TerminalID>
    <CardNum>4000300020001000</CardNum>
    <CardExpMonth>12</CardExpMonth>
    <CardExpYear>22</CardExpYear>
    <TotalAmount>8</TotalAmount>
</ippay>
```
We get the response like this for the successful transaction,
```xml
<ippayResponse>
    <TransactionID>B20191106095946460</TransactionID>
    <ActionCode>000</ActionCode>
    <Approval>TEST01</Approval>
    <ResponseText>APPROVED</ResponseText>
</ippayResponse>
```
##### 3. Response.php
We get the Action code as 000 for the successful transaction. It is also mentioned in isSuccessful() function in this file. 
```php
public function isSuccessful()
    {
        $response = new \SimpleXMLElement($this->response);
        return ($response->ActionCode == "000");
    }
```
We will get some different action code for unsuccessful transaction. Refer page number 7,8 and 9 in this [Document][Actioncode] for more Action codes. 

> Note: These are the acceptable amounts to make successful transaction under test environment in IPpay. Transaction get declined for other than this amount. <br><br>
> 08, 09, 10, 11, 16, 17, 18, 20, 22, 23, 24, 25, 26, 27, 45, 46, 47, 48, 49, 50, 56, 60, 64, 66, 67, 68, 69, 70, 71, 72, 73, 74, 79,86, 87, 88, 89, 95, 97, 98, 99, 100 <br><br>
> If the amount is higher than 100, than those last digits must be the above numbers only

#### Test card details
Card number : 4000 3000 2000 1000 <br>
Expiry date/month : 11/22 <br>
CVV : 123


#### In production environment
* Set TerminalID as 'GOORION00001' and Testmode as 'false' in ippay() function
* Change base_url from localhost to production url in config.php
* Change the database credentials in database.php

#### Software used
* Ubuntu 18.04
* PHP 7.2
* MySQL 14.14
* Composer 1.9.1<br><br><br>
<hr>

Documented by,<br>
Balakrishnan Subramaniyan

>Document version 1.0

[Actioncode]:<https://drive.google.com/file/d/1aEkuhyRPx9qN9Ahq-eWnCDXy_c3zma_X/view?usp=sharing>
[agCl]:<https://github.com/goorionlabs/agriios_1.0.git>
[composer]:<https://getcomposer.org/download/>
