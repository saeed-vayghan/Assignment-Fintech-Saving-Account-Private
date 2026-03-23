
## Preface
You have been appointed as a Solution Architect to a new project spanning across multiple teams. For this project you need to implement a new product - savings accounts. This means that private persons can deposit money into their savings accounts in Alborz Bank and earn interest when their deposits mature.

### Business information:
In order for a person to be eligible to deposit money into their Alborz Bank account they need to be onboarded, identified and we need to gather the required KYC information.

---

### Business requirements:
- Customers: 
    - Individual (private) customers that are residents in the country where they want to open a savings account.
    - Minors, trusts, businesses, governmental entities etc. are not eligible to open a savings account.


- Deposit types:
    - Fixed term deposits: maturity period of 3, 6 or 12 months with fixed interest rate. These deposits can be rolled over i.e. a customer does not withdraw their money and continue their savings contract as an overnight account (calculated with variable interest rate).
    - Overnight savings accounts: offering variable interest rates. The client is allowed to withdraw the money whenever they want and interest is calculated and paid out on a daily basis.

- Transaction types:
    - PYI: Payment received from customer transaction account to fund the deposit account at Alborz Bank, or a later top-up of the invested funds in case of an overnight account
    - PYO: Payout from deposit account at Alborz Bank back to customer’s verified transaction account if deposit is not renewed, or a partial withdrawal in case of an overnight account
    - CAN: Payout from term deposit account at Partner bank back to customer transaction account if a term deposit is canceled irregularly.
    - IBR: Payment of interest to the deposit account at Partner bank, please note that this is just an accounting transaction without any corresponding booking with real money
    - TAX: Source tax deducted, will reduce the eventual payout amount

- Payment import: The payments should be received to a bank account in the market default currency and exported to us as CAMT files on an SFTP server. Based on the reference each of the transactions should be registered to an account and a corresponding transaction should be created into the system. 

- Accounting: run jobs/processes that will:
    - Send back money to client’s accounts that requested them and store them as PYO/CAN transactions based on the type and time of the withdrawal
    - Calculate interest and store them as IBR transactions
    - Calculate taxes and store them as TAX transactions

---

### Legal requirements:
- Identify the client
- Validate client’s bank account
- Check whether the client is on PEP/Sanction lists & raise warnings if found on a list
- Client should pre-fill KYC questionnaire consisting of 5 different questions which can be select or free text questions


## Technical assignment
Design a flow where you implement a new product - fixed term deposit & overnight savings accounts. In your scope you need to consider the onboarding process, underwriting validations as well as internal jobs and processes.
In your technical assignment you are expected to provide:

- Architecture (sequence & process) diagram of the following flows:
    - Onboarding
    - Customer experience - Web application where clients can check their deposit accounts, make withdrawals etc.

- Considerations around best practices:
    - Security
    - Monitoring
    - Logging
    - Developer experience (debugging, deployments etc.)

- Changes that we would have to make to our existing systems to go live with this product assuming that we would like to go live with savings accounts in one year.


## Nice to have:
- Database entity-relationship diagram
- Sample API docs and consideration on how you would structure the APIs
- Questions you would ask stakeholders to get additional information

---

In this assignment please consider the AWS Well-Architected best practices and consider topics
such as monitoring, performance efficiency, security and reliability.