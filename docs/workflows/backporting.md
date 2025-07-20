---
topics: tooling
tags: 
    - "tooling"
---

# Backporting

[Backporting to Stable Branches](https://youtu.be/RxDt5qVO0RM) is the shortest demo. Although it doesn't use kernel trees as examples, the concept is very applicable to kernel maintenance.

For conflict resolution see [Backporting and conflict resolution](https://docs.kernel.org/process/backporting.html). Depending on how the targeted tree is maintained there could be different ways to do it. One possible way is finding all the dependent patches by recursively git-blaming the partent commit of the incoming commit, like what the kernel doc mentioned.

This however might not work if commits in the targeted tree somehow deviates from the upstream too much. Good luck maintaining one then!

## References

### Videos

- [Backporting Linux Kernel patches](https://youtu.be/sBR7R1V2FeA)
- [Shung-Hsi Yu: Backporting BPF: Techniques and Challenges](https://youtu.be/PPM51Mu4fAQ)
- [Backporting to Stable Branches](https://youtu.be/RxDt5qVO0RM)
- [Michal Kubeček: Backporting horror stories](https://youtu.be/zh_pXHEWqIk)
- [LAS16-101: Efficient kernel Backporting](https://youtu.be/-z7_eOR7_Ug)
- [Contributing to the CentOS Stream Kernel - Prarit Bhargava and Don Zickus](https://youtu.be/3J6BjXjc_hQ)

### Links

- [Backporting and conflict resolution](https://docs.kernel.org/process/backporting.html)