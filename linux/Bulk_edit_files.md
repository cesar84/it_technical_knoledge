# Bulk edit files

## Clean up the white spase at the and of each line on each dir and subdir files
egrep -nrl "\s+$" * | awk  '{ print "sed -i \"s/[[:space:]]\\+$//\" " $1 }' | sh