How to make 200MB/day logs out of 20GB/day?

There are couple way of making this happen without loosing critical information. One of which is Sampling.

Sampling doing simple thing - transfer only sample of you data(part) to the logging service. For example some error happening with rate 10 errors/s. Do we need log every error if it the same? Of course not, this will be just waste of resources. So we can log one error per few seconds for example of even minutes.

The more robust solution for example may save additional data which is different, and send it along with the error. Like user_id or session_id

Sampling can be set in a few hours and will start save your money on storing and processing logs, especially when you are using some external SAAS solution.

#logging #economy #savingmoney #savingresources #devops #softwareengineering 
