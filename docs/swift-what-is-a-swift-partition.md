
What is a Swift partition?

## step


Things in Swift are divided up into partitions; you have 2^P partitions, where P is the part power specified at ring creation time. The partition of an object is the first P bits of a hash of its name.

Here's an example:

Let's say you've got an object named "kitten.jpg" in a container "cats" in your Swift account "MY_account", and your object ring has a part power of 16.

Get the secret prefix and suffix (swift_hash_path_prefix and swift_hash_path_suffix) from /etc/swift/swift.conf. Let's say the prefix is "SEC" and the suffix is "RET". Take the object's full name, prepend the prefix, and append the suffix, so you get SEC/MY_account/cats/kitten.jpgRET. Compute the MD5 hash of that string to get 4f3146f77835ff33a6baba807f7f5c1, then take the first 16 bits to get 4f31, or in decimal, 20273.

With that partition number, you can then look in the ring to see which drives hold replicas of that partition, and then contact the relevant nodes to manipulate the object (GET, PUT, DELETE, etc.) This is exactly what the Swift proxy server does to determine which backend(s) to contact to service a request.

## ref

- https://ask.openstack.org/en/question/6766/what-is-a-swift-partition/