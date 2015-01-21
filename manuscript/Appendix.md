# Appendix

## Chapter Status History

## Client-side Chapter

**before Jan 21st, 2015**:
Feel free to use this as a bad example. There are way too many dependencies between namespaces. Also, the application state lives in an atom is not properly protected. I do not feel comfortable with freely passing it around because anyone (including myself) using it from anywhere could accidentally mutilate it, causing the application to fail. Instead, I want to protect it and only hand out immutable data for rendering or whatever other purpose. The required redesign is done in code, it's in **[master](https://github.com/matthiasn/BirdWatch)** on GitHub. Chapter rewrites and archtectural drawings due very soon.
