---
layout: post
title: Designing a SQL Parser - Introduction
---



# Code sample


{% highlight python linenos %}
#!/usr/bin/python
import os
import sys

def main():
	sum = 0

	with open(sys.argv[1]) as f:
		for line in f:
			fields = line.rstrip().split('|')
			
			quantity = float(fields[4])
			eprice = float(fields[5])
			discount = float(fields[6])

			if(discount > 0.05 and discount < 0.07 and quantity < 24):
				sum += eprice * discount;

	print(sum)


if __name__ == "__main__":
	main()
{% endhighlight %}