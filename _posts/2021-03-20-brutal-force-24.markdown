---
layout: post
title:  "24 game brutal force algorithm"
date:   2021-03-20 10:00:00 +0800
categories: algorithm
---

Given 4 integer numbers, the algothrim needs to tell if any combination of '+', '-', '*', '/' and '()' can calculate number 24.

Below is a brutal force algorithm:

```python
import sys
import pprint


def main(nums):
    nums4 = list(map(int, nums))
    nums_combinations = []
    for n1 in nums4:
        nums3 = nums4[:]
        nums3.remove(n1)
        for n2 in nums3:
            nums2 = nums3[:]
            nums2.remove(n2)
            for n3 in nums2:
                nums1 = nums2[:]
                nums1.remove(n3)
                n4 = nums1[0]
                nums_combinations.append([n1, n2, n3, n4])

    symbols_combinations = []
    symbols = ['+', '-', '*', '/']
    for s1 in symbols:
        for s2 in symbols:
            for s3 in symbols:
                symbols_combinations.append([s1, s2, s3])

    possible_calculations = []
    for n1, n2, n3, n4 in nums_combinations:
        for s1, s2, s3 in symbols_combinations:
            cal_strs = []
            cal_strs.append(f'{n1}{s1}{n2}{s2}{n3}{s3}{n4}')
            cal_strs.append(f'({n1}{s1}{n2}){s2}{n3}{s3}{n4}')
            cal_strs.append(f'{n1}{s1}({n2}{s2}{n3}){s3}{n4}')
            cal_strs.append(f'{n1}{s1}{n2}{s2}({n3}{s3}{n4})')
            cal_strs.append(f'({n1}{s1}{n2}{s2}{n3}){s3}{n4}')
            cal_strs.append(f'{n1}{s1}({n2}{s2}{n3}{s3}{n4})')
            cal_strs.append(f'({n1}{s1}{n2}{s2}{n3}){s3}{n4}')
            cal_strs.append(f'{n1}{s1}({n2}{s2}{n3}{s3}{n4})')
            cal_strs.append(f'({n1}{s1}{n2}){s2}({n3}{s3}{n4})')
            cal_strs.append(f'(({n1}{s1}{n2}){s2}{n3}){s3}{n4}')
            cal_strs.append(f'({n1}{s1}({n2}{s2}{n3})){s3}{n4}')
            cal_strs.append(f'{n1}{s1}(({n2}{s2}{n3}){s3}{n4})')
            cal_strs.append(f'{n1}{s1}({n2}{s2}({n3}{s3}{n4}))')
            for cal_str in cal_strs:
                try:
                    if eval(cal_str) == 24:
                        possible_calculations.append(cal_str)
                except:
                    pass

    pprint.pprint(possible_calculations)

if __name__ == '__main__':

    main(sys.argv[1:5])

```