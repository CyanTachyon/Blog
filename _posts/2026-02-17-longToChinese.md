---
title: Int64转中文
date: 2026-02-17 23:30:00 +0800
tags:
  - develop
  - ai
  - llm
author: CyanTachyon
categories:
  - develop
  - ai
math: true
---

弄了一个题目来测试市面上的LLM的能力：
> 给定一个int64，将其转换为中文数字表示的字符串。（编译语言自选）

<details markdown="1" inline>
<summary>如果你小学没有毕业，不知道中文数字的读法</summary>

- 基本数字：0-9 分别对应 零、一、二、三、四、五、六、七、八、九
- 数位单位：从个位开始，每四位一个单位，分别是 万、亿、兆、京（10^4、10^8、10^12、10^16）
- 读法规则：
  - 从最高位开始读，每四位为一段，读出该段的数字和单位（如果该段不为零）。例如，1234567890 可以分为 12,3456,7890 3段，分别读作 "十二亿"、"三千四百五十六万"、"七千八百九十"
  - 每段内部的读法：千位、百位、十位、个位
    - 千位：如果不为零，读作 "X千"
    - 百位：如果不为零，读作 "X百"
    - 十位：如果不为零，读作 "X十"，特别的，如果在整个数字的最前面时，那应当读作 "十" 而不是 "一十"。仅当在「整个数字的最前面时」才读作 "十"，其他位置的 "10" 都应读作 "一十"。例如10010应读作 "一万零一十"，而不是 "一万零十"。
    - 个位：如果不为零，读作 "X"
  - 零的处理：
    - 如果某位为零，且其不是该段的最后一位，且其下一位非零，则读作 "零"。
    - 如果某段全为零，且其下一段开头非零，则读作零。
    - 其余情况不读零。
  - 负数：在结果前加 "负"。

</details>

同时放在了 [luogo U661323](https://www.luogu.com.cn/problem/U661323)

这个题目看起来非常简单，但实际上有非常多细节需要处理，无论是人还是LLM，想要一次写对还是有一定难度的。

以下是一些模型的测试表现：

<details markdown="1" blk>
<summary>DeepSeek 2026-02-16日官网上的未知模型</summary>

> 思考时间：较长，约7分钟。  
> 选择的语言：CPP  
> 结果：一次性通过  

这其实有点出乎意料，毕竟ds的coding表现并不算特别好。

对于long最小值，ds的做法是将其转为unsigned long，从而避免了溢出问题。

<details markdown="1" inline>
<summary>代码</summary>

```cpp
#include <iostream>
#include <string>
#include <vector>

using namespace std;

const string digits[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
const string units[] = {"", "万", "亿", "兆", "京"}; // 单位，对应个、万、亿、兆、京

// 将一段4位数字（0除外）转换为中文数字串，isFirstSegment表示该段是否为整个数的最高段
string segmentToString(int val, bool isFirstSegment) {
    int q = val / 1000;               // 千位
    int b = (val / 100) % 10;          // 百位
    int s = (val / 10) % 10;           // 十位
    int g = val % 10;                  // 个位
    int digs[4] = {q, b, s, g};
    string res;
    bool needZero = false;             // 是否需要插入一个“零”
    bool firstNonZero = true;          // 该段内是否还没遇到非零位

    for (int i = 0; i < 4; ++i) {
        int d = digs[i];
        if (d == 0) {
            // 当前位为0，且不是最后一位，并且不是最高段的首个前导零，则考虑标记needZero
            if (i < 3) {
                // 排除最高段且是第一个非零位之前的情况（即前导零）
                if (!(isFirstSegment && firstNonZero) && digs[i + 1] != 0) {
                    needZero = true;
                }
            }
        } else {
            if (needZero) {
                res += "零";
                needZero = false;
            }
            // 处理十位的特殊读法：整个数的最高位且十位为1时读作“十”
            if (i == 2 && d == 1 && isFirstSegment && firstNonZero) {
                res += "十";
            } else {
                res += digits[d];
                if (i == 0) res += "千";
                else if (i == 1) res += "百";
                else if (i == 2) res += "十";
                // 个位不加单位
            }
            firstNonZero = false;
        }
    }
    return res;
}

int main() {
    long long n;
    cin >> n;

    if (n == 0) {
        cout << "零" << endl;
        return 0;
    }

    string result;
    if (n < 0) {
        result += "负";
    }

    // 取绝对值，用 unsigned long long 避免最小值溢出
    unsigned long long u = (n < 0) ? - (unsigned long long)n : (unsigned long long)n;

    // 从低位到高位每四位一组，存入vector
    vector<int> segs;
    while (u > 0) {
        segs.push_back(u % 10000);
        u /= 10000;
    }
    // 此时 segs[0] 是个位段，segs[1] 是万位段，依此类推

    bool firstNonZeroSegment = true;   // 是否还没遇到非零段（用于判断最高段）
    bool pendingZero = false;          // 是否需要插入段间零

    // 从高位段向低位段遍历
    for (int i = (int)segs.size() - 1; i >= 0; --i) {
        int val = segs[i];
        if (val == 0) {
            // 全零段：如果前面已经有非零段，则标记需要插入零
            if (!firstNonZeroSegment) {
                pendingZero = true;
            }
        } else {
            // 处理当前非零段
            string segStr = segmentToString(val, firstNonZeroSegment);
            if (pendingZero) {
                // 如果该段内部不以“零”开头，则需先加一个段间零
                if (segStr.find("零") != 0) {
                    result += "零";
                }
                pendingZero = false;
            }
            result += segStr;
            if (i > 0) { // 不是个位段，加上单位
                result += units[i];
            }
            firstNonZeroSegment = false;
        }
    }

    cout << result << endl;
    return 0;
}
```

</details>

</details>

<details markdown="1" blk>
<summary>ChatGPT-5.2 XHigh</summary>

> 思考时间：较长，和ds不相上下  
> 选择的语言：CPP  
> 结果：一次性通过  

ChatGPT-5.2的表现和ds非常接近，都是一次性通过，且思考时间也差不多。对于long最小值的问题，gpt将其转为了int128，再转成了unsigned long。我认为在这个处理上不如ds的做法直接转为unsigned long，完全无需使用128位整数。

<details markdown="1" inline>
<summary>代码</summary>

```cpp
#include <bits/stdc++.h>
using namespace std;

static const vector<string> CN_NUM = {u8"零",u8"一",u8"二",u8"三",u8"四",u8"五",u8"六",u8"七",u8"八",u8"九"};
static const vector<string> SMALL_UNIT = {u8"千", u8"百", u8"十", u8""};
static const vector<string> BIG_UNIT = {u8"", u8"万", u8"亿", u8"兆", u8"京"};

string convert4(int x, bool isTopGroup) { // 1..9999
    int d[4] = { x / 1000, (x / 100) % 10, (x / 10) % 10, x % 10 };
    string res;
    bool started = false;   // 段内是否已经输出过非零
    bool zeroPend = false;  // 段内是否需要补一个“零”

    for (int i = 0; i < 4; i++) {
        int digit = d[i];
        if (digit == 0) {
            if (started) zeroPend = true; // 可能需要在下一个非零前输出“零”
        } else {
            if (zeroPend) {
                res += CN_NUM[0];
                zeroPend = false;
            }
            // 十位特判：只有“整个数字最前面”才把 10~19 读作“十X”
            if (i == 2 && digit == 1 && !started && isTopGroup) {
                res += SMALL_UNIT[i]; // "十"
            } else {
                res += CN_NUM[digit];
                res += SMALL_UNIT[i];
            }
            started = true;
        }
    }
    return res;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    long long x;
    cin >> x;

    if (x == 0) {
        cout << u8"零";
        return 0;
    }

    bool neg = (x < 0);
    __int128 t = x;
    if (neg) t = -t; // 用 __int128 避免 LLONG_MIN 取负溢出
    unsigned long long u = (unsigned long long)t;

    // 按 10000 进制拆段：group[0] 是最低段
    vector<int> group;
    while (u > 0) {
        group.push_back((int)(u % 10000));
        u /= 10000;
    }

    string ans;
    bool needZero = false; // 前面出现过“整段为 0”，下一段非零时需要输出一个“零”

    for (int i = (int)group.size() - 1; i >= 0; --i) {
        int g = group[i];
        if (g == 0) {
            if (!ans.empty()) needZero = true;
            continue;
        }

        if (!ans.empty() && (needZero || g < 1000)) {
            ans += u8"零";
        }
        needZero = false;

        ans += convert4(g, ans.empty()); // ans.empty() 说明这是最高非零段
        ans += BIG_UNIT[i];
    }

    if (neg) ans = string(u8"负") + ans;
    cout << ans;
    return 0;
}
```

</details>

</details>

<details markdown="1" blk>
<summary>GLM5</summary>

> 思考时间：较长，和前两个模型差不多，两次修改思考时间也差不多都很长  
> 选择的语言：CPP  
> 结果：修改两次才通过  

GLM5属实没有达到我的预期，作为一个专注coding的模型，处理的不是很好。

第一次提交代码，「零」的处理有很大问题，例如1001会被读成「一千零零一」。修正一次后，仍有一个测试点错误（100000001读成「一亿零零一」）。修正后才完全通过。

<details markdown="1" inline>
<summary>第一次提交的代码</summary>

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <climits>

using namespace std;

// 数字映射
string digit_map[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
// 单位映射：索引对应段的级别，0为个位段，1为万段...
string unit_map[] = {"", "万", "亿", "兆", "京"};
// 段内单位
string inner_unit_map[] = {"", "十", "百", "千"};

// 将一个4位整数段转换为中文读法
// val: 0-9999
// is_first_segment: 是否是整个数字的最高有效段（用于处理"一十"的特例）
string convert_segment(unsigned int val, bool is_first_segment) {
    string res = "";
    
    // 提取各位数字
    int digits[4] = {0};
    digits[0] = val % 10;        // 个位
    digits[1] = (val / 10) % 10; // 十位
    digits[2] = (val / 100) % 10; // 百位
    digits[3] = (val / 1000) % 10; // 千位
    
    // 从千位开始处理
    // 在段内，只有当还没输出过任何数字时，才处于“段首”状态，需要考虑“一十”变“十”的规则
    bool segment_start = true; 

    for (int i = 3; i >= 0; i--) {
        int d = digits[i];
        
        if (d == 0) {
            // 如果当前位是0，且不是段的最后一位，且下一位非零，则读“零”
            // 注意：如果我们在段首，前面的0是不读的（例如 0012 读作 十二，而不是 零零十二）
            if (!segment_start) {
                // 检查后面是否还有非零数字
                bool has_next_nonzero = false;
                for (int j = i - 1; j >= 0; j--) {
                    if (digits[j] != 0) {
                        has_next_nonzero = true;
                        break;
                    }
                }
                if (has_next_nonzero) {
                    res += "零";
                }
            }
        } else {
            // 处理非零数字
            
            // “一十”特判：
            // 规则：仅在“整个数字的最前面”时，"10"读作"十"。
            // 表现：当前是段首(is_first_segment && segment_start)，且当前是十位(i==1)，且数字是1。
            if (is_first_segment && segment_start && i == 1 && d == 1) {
                // 这种情况下，数字"1"省略，只读单位"十"
                res += "十";
            } else {
                // 正常读法：数字 + 单位
                res += digit_map[d] + inner_unit_map[i];
            }
            
            segment_start = false; // 只要输出了非零内容，段首状态结束
        }
    }
    
    return res;
}

int main() {
    // 优化输入输出效率
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    long long x;
    if (!(cin >> x)) return 0;

    bool neg = false;
    unsigned long long n;

    // 处理负数
    if (x < 0) {
        neg = true;
        // 处理 INT64_MIN 的溢出问题
        // 直接取负会溢出，因为 int64 范围不对称
        if (x == LLONG_MIN) {
            n = 1ULL << 63; // 2^63
        } else {
            n = -x;
        }
    } else {
        n = x;
    }

    // 特判 0
    if (n == 0) {
        cout << "零" << endl;
        return 0;
    }

    // 分段：将数字按4位一组拆分
    vector<unsigned int> segments;
    while (n > 0) {
        segments.push_back(n % 10000);
        n /= 10000;
    }

    string ans = "";
    bool is_global_start = true; // 标记是否刚开始输出（用于判断是否是最高有效段）

    // 从最高段开始遍历
    for (int i = segments.size() - 1; i >= 0; i--) {
        unsigned int val = segments[i];
        
        if (val == 0) {
            // 如果当前段全为0，且下一段并非全为0（或者更低位非零），则需要补零
            // 检查低位是否有非零段
            bool has_next_nonzero_segment = false;
            for (int j = i - 1; j >= 0; j--) {
                if (segments[j] != 0) {
                    has_next_nonzero_segment = true;
                    break;
                }
            }
            
            // 如果之前已经输出过内容（非开头），且低位还有内容，则补“零”
            if (!is_global_start && has_next_nonzero_segment) {
                ans += "零";
            }
        } else {
            // 当前段非零，转换段内数字
            // 传入 is_global_start 用于处理 "一十" -> "十" 的特例
            ans += convert_segment(val, is_global_start);
            
            // 添加段单位（万、亿...），个位段不加单位
            if (i > 0) {
                ans += unit_map[i];
            }
            
            is_global_start = false; // 已经输出了内容，不再是开头
        }
    }

    // 添加负号
    if (neg) {
        ans = "负" + ans;
    }

    cout << ans << endl;

    return 0;
}
```

</details>

<details markdown="1" inline>
<summary>第二次提交的代码</summary>

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <climits>

using namespace std;

// 数字映射
string digit_map[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
// 单位映射
string unit_map[] = {"", "万", "亿", "兆", "京"};
// 段内单位
string inner_unit_map[] = {"", "十", "百", "千"};

// 将一个4位整数段转换为中文读法
// val: 0-9999
// is_first_segment: 是否是整个数字的最高有效段（用于处理前导零和"一十"的特例）
string convert_segment(unsigned int val, bool is_first_segment) {
    string res = "";
    
    // 提取各位数字
    int digits[4] = {0};
    digits[0] = val % 10;        // 个位
    digits[1] = (val / 10) % 10; // 十位
    digits[2] = (val / 100) % 10; // 百位
    digits[3] = (val / 1000) % 1000; // 千位
    
    bool segment_start = true; // 标记是否在当前段的开头

    for (int i = 3; i >= 0; i--) {
        int d = digits[i];
        
        if (d == 0) {
            // 规则：如果某位为零，且其不是该段的最后一位，且其下一位非零，则读作 "零"。
            // 注意：如果是整个数字的最高段的开头，前导零不读。
            
            // 如果是在段的开头且是最高段，直接跳过
            if (is_first_segment && segment_start) {
                continue;
            }
            
            // 判断下一位是否非零 (i > 0 保证不是最后一位)
            if (i > 0 && digits[i-1] != 0) {
                res += "零";
            }
        } else {
            // 处理非零数字
            
            // "一十"特判：
            // 规则：仅当在「整个数字的最前面时」"10"读作"十"。
            // 条件：最高段 && 段首 && 是十位(i==1) && 数字是1
            if (is_first_segment && segment_start && i == 1 && d == 1) {
                res += "十";
            } else {
                // 正常读法：数字 + 单位
                res += digit_map[d] + inner_unit_map[i];
            }
            
            segment_start = false; // 只要有非零输出，段首状态结束
        }
    }
    
    return res;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    long long x;
    if (!(cin >> x)) return 0;

    bool neg = false;
    unsigned long long n;

    if (x < 0) {
        neg = true;
        if (x == LLONG_MIN) {
            n = 1ULL << 63;
        } else {
            n = -x;
        }
    } else {
        n = x;
    }

    if (n == 0) {
        cout << "零" << endl;
        return 0;
    }

    vector<unsigned int> segments;
    while (n > 0) {
        segments.push_back(n % 10000);
        n /= 10000;
    }

    string ans = "";
    bool is_global_start = true;

    for (int i = segments.size() - 1; i >= 0; i--) {
        unsigned int val = segments[i];
        
        if (val == 0) {
            // 全零段处理：如果下一段并非全为零，则读作零
            bool has_next_nonzero_segment = false;
            for (int j = i - 1; j >= 0; j--) {
                if (segments[j] != 0) {
                    has_next_nonzero_segment = true;
                    break;
                }
            }
            
            // 只有在非开头情况下才补零，避免输出 "零..."
            if (!is_global_start && has_next_nonzero_segment) {
                ans += "零";
            }
        } else {
            string part = convert_segment(val, is_global_start);
            
            // 跨段零合并：
            // 如果之前的结果以"零"结尾，且当前段以"零"开头（例如 1,0005 中的 0005 -> "零五"）
            // 则合并为一个"零"。
            if (!ans.empty() && ans.back() == '零' && !part.empty() && part.front() == '零') {
                part.erase(0, 3); // 移除 part 开头的 "零" (UTF-8 3字节)
            }
            
            ans += part;
            
            if (i > 0) {
                ans += unit_map[i];
            }
            
            is_global_start = false;
        }
    }

    if (neg) {
        ans = "负" + ans;
    }

    cout << ans << endl;

    return 0;
}
```

</details>

<details markdown="1" inline>
<summary>最终代码</summary>

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <climits>

using namespace std;

// 数字映射
string digit_map[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
// 单位映射
string unit_map[] = {"", "万", "亿", "兆", "京"};
// 段内单位
string inner_unit_map[] = {"", "十", "百", "千"};

// 将一个4位整数段转换为中文读法
// val: 0-9999
// is_first_segment: 是否是整个数字的最高有效段（用于处理"一十"的特例）
string convert_segment(unsigned int val, bool is_first_segment) {
    string res = "";
    
    // 提取各位数字
    int digits[4] = {0};
    digits[0] = val % 10;        // 个位
    digits[1] = (val / 10) % 10; // 十位
    digits[2] = (val / 100) % 10; // 百位
    digits[3] = (val / 1000) % 10; // 千位
    
    bool segment_start = true; // 标记是否在当前段的开头

    for (int i = 3; i >= 0; i--) {
        int d = digits[i];
        
        if (d == 0) {
            // 规则：如果某位为零，且其不是该段的最后一位，且其下一位非零，则读作 "零"。
            // 注意：如果是整个数字的最高段的开头，前导零不读。
            
            // 如果是在段的开头且是最高段，直接跳过
            if (is_first_segment && segment_start) {
                continue;
            }
            
            // 判断下一位是否非零 (i > 0 保证不是最后一位)
            if (i > 0 && digits[i-1] != 0) {
                res += "零";
            }
        } else {
            // 处理非零数字
            
            // "一十"特判：
            // 规则：仅当在「整个数字的最前面时」"10"读作"十"。
            // 条件：最高段 && 段首 && 是十位(i==1) && 数字是1
            if (is_first_segment && segment_start && i == 1 && d == 1) {
                res += "十";
            } else {
                // 正常读法：数字 + 单位
                res += digit_map[d] + inner_unit_map[i];
            }
            
            segment_start = false; // 只要有非零输出，段首状态结束
        }
    }
    
    return res;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    long long x;
    if (!(cin >> x)) return 0;

    bool neg = false;
    unsigned long long n;

    if (x < 0) {
        neg = true;
        if (x == LLONG_MIN) {
            n = 1ULL << 63;
        } else {
            n = -x;
        }
    } else {
        n = x;
    }

    if (n == 0) {
        cout << "零" << endl;
        return 0;
    }

    vector<unsigned int> segments;
    while (n > 0) {
        segments.push_back(n % 10000);
        n /= 10000;
    }

    string ans = "";
    bool is_global_start = true;

    for (int i = segments.size() - 1; i >= 0; i--) {
        unsigned int val = segments[i];
        
        if (val == 0) {
            // 全零段处理
            // 寻找下一个非零段
            int next_non_zero_idx = -1;
            for (int j = i - 1; j >= 0; j--) {
                if (segments[j] != 0) {
                    next_non_zero_idx = j;
                    break;
                }
            }
            
            // 只有在非开头情况下，且后续还有非零段，才考虑补零
            if (!is_global_start && next_non_zero_idx != -1) {
                // 检查下一个非零段是否小于 1000
                // 如果小于 1000，则该段开头必然是"零"（段内逻辑会处理），这里不需要额外补零
                // 例如：100000001 -> 1,0000,0001。0000段不补零，由0001段读出"零一"
                if (segments[next_non_zero_idx] >= 1000) {
                    ans += "零";
                }
            }
        } else {
            string part = convert_segment(val, is_global_start);
            
            // 这里不需要再判断 part[0] == '零' 进行消重了，
            // 因为我们已经在全零段逻辑中避免了重复。
            
            ans += part;
            
            if (i > 0) {
                ans += unit_map[i];
            }
            
            is_global_start = false;
        }
    }

    if (neg) {
        ans = "负" + ans;
    }

    cout << ans << endl;

    return 0;
}
```

</details>

</details>

<details markdown="1" blk>
<summary>Kimi K2.5</summary>

> 思考时间：较长和前几个模型差不多  
> 选择的语言：CPP  
> 结果：修正一次后才通过  

其第一次错误是在处理「一十」的特例时，不只把「一十」的「一」去掉了，还把20～90的数字也去掉了，例如「二十」被读成了「十」。修正后才完全通过。

<details markdown="1" inline>
<summary>第一次提交的代码</summary>

```cpp
#include <bits/stdc++.h>
using namespace std;

const string digits[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
const string units[] = {"", "万", "亿", "兆", "京"};  // 10^0, 10^4, 10^8, 10^12, 10^16

// 转换4位数字（0-9999）
// isHighest: 是否是整个数字的最高段（用于处理"十"的特殊读法）
// needZeroPrefix: 是否需要在开头补"零"（前一段为0导致）
string convert4(int num, bool isHighest, bool needZeroPrefix) {
    if (num == 0) return "";
    
    string res;
    int qian = num / 1000;
    int bai = (num / 100) % 10;
    int shi = (num / 10) % 10;
    int ge = num % 10;
    
    bool needZero = needZeroPrefix;  // 标记是否需要在下一位前加"零"
    
    // 千位
    if (qian != 0) {
        if (needZero) res += "零";
        res += digits[qian] + "千";
        needZero = false;
    } else if (!isHighest && (bai != 0 || shi != 0 || ge != 0)) {
        // 千位为0，不是最高段，且后面还有非零数字，需要读"零"
        needZero = true;
    }
    
    // 百位
    if (bai != 0) {
        if (needZero) res += "零";
        res += digits[bai] + "百";
        needZero = false;
    } else if (qian != 0 && (shi != 0 || ge != 0)) {
        // 千位非零，百位为0，后面还有非零数字，需要读"零"
        needZero = true;
    }
    
    // 十位
    if (shi != 0) {
        if (needZero) res += "零";
        // 特殊规则：如果在整个数字的最前面（即没有千位和百位），读"十"而非"一十"
        if (qian == 0 && bai == 0 && isHighest) {
            res += "十";
        } else {
            res += digits[shi] + "十";
        }
        needZero = false;
    } else if (bai != 0 && ge != 0) {
        // 百位非零，十位为0，个位非零，需要读"零"
        needZero = true;
    }
    
    // 个位
    if (ge != 0) {
        if (needZero) res += "零";
        res += digits[ge];
    }
    
    return res;
}

string convert(long long n) {
    if (n == 0) return "零";
    
    bool neg = false;
    unsigned long long un;
    
    if (n < 0) {
        neg = true;
        // 处理 LLONG_MIN (-2^63)，避免溢出
        un = (unsigned long long)n;
        un = ~un + 1ULL;  // 取绝对值（补码）
    } else {
        un = (unsigned long long)n;
    }
    
    // 按每4位分段（个、万、亿、兆、京）
    vector<int> segs;
    while (un > 0) {
        segs.push_back((int)(un % 10000ULL));
        un /= 10000ULL;
    }
    
    string res;
    bool needZero = false;  // 标记下一段是否需要前导"零"（当前段为0但下一段非0）
    
    for (int i = (int)segs.size() - 1; i >= 0; --i) {
        if (segs[i] == 0) {
            // 当前段为0，如果下一段非0，则需要读"零"
            if (i > 0 && segs[i-1] != 0) {
                needZero = true;
            }
        } else {
            bool isHighest = (i == (int)segs.size() - 1);
            string segStr = convert4(segs[i], isHighest, needZero);
            res += segStr + units[i];
            needZero = false;
        }
    }
    
    if (neg) res = "负" + res;
    return res;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    long long n;
    if (cin >> n) {
        cout << convert(n) << '\n';
    }
    return 0;
}
```

</details>

<details markdown="1" inline>
<summary>第二次提交的代码</summary>

```cpp
#include <bits/stdc++.h>
using namespace std;

const string digits[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
const string units[] = {"", "万", "亿", "兆", "京"};

// 转换4位数字（0-9999）
string convert4(int num, bool isHighest, bool needZeroPrefix) {
    if (num == 0) return "";
    
    string res;
    int qian = num / 1000;
    int bai = (num / 100) % 10;
    int shi = (num / 10) % 10;
    int ge = num % 10;
    
    bool needZero = needZeroPrefix;
    
    // 千位
    if (qian != 0) {
        if (needZero) res += "零";
        res += digits[qian] + "千";
        needZero = false;
    } else if (!isHighest && (bai != 0 || shi != 0 || ge != 0)) {
        needZero = true;
    }
    
    // 百位
    if (bai != 0) {
        if (needZero) res += "零";
        res += digits[bai] + "百";
        needZero = false;
    } else if (qian != 0 && (shi != 0 || ge != 0)) {
        needZero = true;
    }
    
    // 十位
    if (shi != 0) {
        if (needZero) res += "零";
        // 特殊规则：仅当在「整个数字的最前面」且十位为1时，才读作 "十" 而不是 "一十"
        if (qian == 0 && bai == 0 && isHighest && shi == 1) {
            res += "十";
        } else {
            res += digits[shi] + "十";
        }
        needZero = false;
    } else if (bai != 0 && ge != 0) {
        needZero = true;
    }
    
    // 个位
    if (ge != 0) {
        if (needZero) res += "零";
        res += digits[ge];
    }
    
    return res;
}

string convert(long long n) {
    if (n == 0) return "零";
    
    bool neg = false;
    unsigned long long un;
    
    if (n < 0) {
        neg = true;
        un = (unsigned long long)n;
        un = ~un + 1ULL;
    } else {
        un = (unsigned long long)n;
    }
    
    vector<int> segs;
    while (un > 0) {
        segs.push_back((int)(un % 10000ULL));
        un /= 10000ULL;
    }
    
    string res;
    bool needZero = false;
    
    for (int i = (int)segs.size() - 1; i >= 0; --i) {
        if (segs[i] == 0) {
            if (i > 0 && segs[i-1] != 0) {
                needZero = true;
            }
        } else {
            bool isHighest = (i == (int)segs.size() - 1);
            string segStr = convert4(segs[i], isHighest, needZero);
            res += segStr + units[i];
            needZero = false;
        }
    }
    
    if (neg) res = "负" + res;
    return res;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    long long n;
    if (cin >> n) {
        cout << convert(n) << '\n';
    }
    return 0;
}
```

</details>

</details>

<details markdown="1" blk>
<summary>Gemini 3 Pro High</summary>

> 思考时间：极短，几乎没有思考  
> 选择的语言：CPP  
> 结果：修改一次后才通过  

Gemini也是多个零的时候没有处理好，100000001被读成了「一亿零零一」，修正后才完全通过。
<details markdown="1" inline>
<summary>第一次提交的代码</summary>

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

using namespace std;

typedef long long ll;
typedef __int128_t int128; // 使用 int128 防止 abs(-2^63) 溢出

string nums[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
string units[] = {"", "十", "百", "千"};
string sections[] = {"", "万", "亿", "兆", "京"};

// 处理 4 位以内的数字
string process_4_digits(int n, bool is_leading_segment, bool is_entire_start) {
    if (n == 0) return "";
    
    string res = "";
    int d[4]; // 千, 百, 十, 个
    d[0] = n / 1000;
    d[1] = (n / 100) % 10;
    d[2] = (n / 10) % 10;
    d[3] = n % 10;

    bool last_was_zero = false;
    bool has_started = false;

    for (int i = 0; i < 4; ++i) {
        if (d[i] != 0) {
            if (last_was_zero) {
                res += nums[0];
            }
            
            // 特殊处理 "一十"：仅在整个数字的最开头且是十位时读作 "十"
            if (i == 2 && d[i] == 1 && is_entire_start && !has_started && d[0] == 0 && d[1] == 0) {
                res += units[1];
            } else {
                res += nums[d[i]] + units[3 - i];
            }
            
            has_started = true;
            last_was_zero = false;
        } else {
            if (has_started) {
                last_was_zero = true;
            }
        }
    }
    return res;
}

string solve() {
    ll input_n;
    if (!(cin >> input_n)) return "";
    if (input_n == 0) return "零";

    string prefix = "";
    int128 n = input_n;
    if (n < 0) {
        prefix = "负";
        n = -n;
    }

    // 分段：每4位一段
    vector<int> parts;
    while (n > 0) {
        parts.push_back((int)(n % 10000));
        n /= 10000;
    }

    string result = "";
    bool has_prev_nonzero = false;

    for (int i = parts.size() - 1; i >= 0; --i) {
        int current_part = parts[i];
        
        if (current_part != 0) {
            // 如果这一段不是第一段，且它小于 1000 (即千位为0)，
            // 或者前一段全为0但后面还有数，需要补 "零"
            if (has_prev_nonzero && current_part < 1000) {
                result += "零";
            }

            result += process_4_digits(current_part, i == (int)parts.size() - 1, !has_prev_nonzero);
            result += sections[i];
            has_prev_nonzero = true;
        } else {
            // 如果这一段全为零，且后面还有非零段，且之前已经有非零段了
            // 规则：如果某段全为零，且其下一段并非全为零，则读作零
            if (has_prev_nonzero) {
                // 检查后面是否还有非零段
                bool remaining_nonzero = false;
                for (int j = i - 1; j >= 0; --j) {
                    if (parts[j] != 0) {
                        remaining_nonzero = true;
                        break;
                    }
                }
                if (remaining_nonzero) {
                    // 只有当 result 末尾不是 "零" 时才添加
                    if (result.size() >= 3 && result.substr(result.size() - 3) != "零") {
                        result += "零";
                    }
                    // 重置 has_prev_nonzero 为 true，但不再读这一段的单位
                }
            }
        }
    }

    // 清理可能由于分段逻辑产生的重复“零”
    // 虽然逻辑上尽量避免了，但中文读法中“零”的规则较碎
    return prefix + result;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(0);
    cout << solve() << endl;
    return 0;
}
```

</details>

<details markdown="1" inline>
<summary>第二次提交的代码</summary>

```cpp
#include <iostream>
#include <vector>
#include <string>

using namespace std;

typedef __int128_t int128; // 处理 int64 最小值溢出

string nums[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
string units[] = {"", "十", "百", "千"};
string sections[] = {"", "万", "亿", "兆", "京"};

// 处理 4 位以内的数字，增加 start_of_whole 标记处理“一十”规则
string process_4_digits(int n, bool start_of_whole) {
    string res = "";
    int d[4];
    d[0] = n / 1000;         // 千
    d[1] = (n / 100) % 10;   // 百
    d[2] = (n / 10) % 10;    // 十
    d[3] = n % 10;           // 个

    bool has_started = false;
    for (int i = 0; i < 4; ++i) {
        if (d[i] != 0) {
            // 段内零：如果当前位不是段内第一位，且前一位是0，补“零”
            if (has_started && d[i-1] == 0) {
                res += nums[0];
            }
            
            // “一十”特判：仅在整个数字的最开头且是“1X”时读作“十”
            if (i == 2 && d[i] == 1 && start_of_whole && !has_started && d[0] == 0 && d[1] == 0) {
                res += units[1];
            } else {
                res += nums[d[i]] + units[3 - i];
            }
            has_started = true;
        }
    }
    return res;
}

void solve() {
    long long input_n;
    if (!(cin >> input_n)) return;
    if (input_n == 0) {
        cout << "零" << endl;
        return;
    }

    int128 n = input_n;
    if (n < 0) {
        cout << "负";
        n = -n;
    }

    vector<int> parts;
    while (n > 0) {
        parts.push_back((int)(n % 10000));
        n /= 10000;
    }

    string result = "";
    bool has_prev_nonzero = false; // 标记之前是否已经有过非零段
    bool gap_zero = false;         // 标记段间是否存在全零段

    for (int i = (int)parts.size() - 1; i >= 0; --i) {
        if (parts[i] == 0) {
            // 规则：如果某段全为零，且之前有数字，标记需要补零
            if (has_prev_nonzero) gap_zero = true;
        } else {
            // 在读入非零段之前，检查是否需要补“零”
            // 情况A：之前有全零段 (gap_zero)
            // 情况B：当前段不足四位（千位为0），且之前有数字
            if (gap_zero || (has_prev_nonzero && parts[i] < 1000)) {
                result += "零";
                gap_zero = false; // 补完零后重置标记
            }

            // 只有当这是整个数字的第一个非零段时，才传递 true 给 process_4_digits
            result += process_4_digits(parts[i], !has_prev_nonzero);
            result += sections[i];
            has_prev_nonzero = true;
        }
    }
    cout << result << endl;
}

int main() {
    solve();
    return 0;
}
```

</details>

</details>

<details markdown="1" blk>
<summary>Qwen3.5-397B-A17B</summary>

> 思考时间：极短，基本没有思考  
> 选择的语言：CPP  
> 结果：修正一次后才通过  

第一次代码仅错误一个测试点，把105000读做了「十万零五千」（正确读法是「十万零五」），修正后才完全通过。
<details markdown="1" inline>
<summary>第一次提交的代码</summary>

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

using namespace std;

// 基本数字映射
const string DIGITS[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
// 段内单位 (个，十，百，千) -> 对应 p % 4
const string INTERNAL_UNITS[] = {"", "十", "百", "千"};
// 段间单位 (无，万，亿，兆，京) -> 对应 p / 4
const string BLOCK_UNITS[] = {"", "万", "亿", "兆", "京"};

int main() {
    // 优化 I/O 操作
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    string s;
    if (!(cin >> s)) return 0;

    bool is_negative = false;
    if (s[0] == '-') {
        is_negative = true;
        s = s.substr(1);
    }

    // 处理 0 的情况
    // 去除前导零，但保留最后一个零如果是全零
    bool all_zero = true;
    for (char c : s) {
        if (c != '0') {
            all_zero = false;
            break;
        }
    }

    if (all_zero) {
        cout << "零" << endl;
        return 0;
    }

    // 去除前导零
    size_t first_nonzero = s.find_first_not_of('0');
    if (first_nonzero != string::npos) {
        s = s.substr(first_nonzero);
    } else {
        // 理论上上面 all_zero 已经处理了全 0，这里以防万一
        cout << "零" << endl;
        return 0;
    }

    // 填充长度到 4 的倍数
    int len = s.length();
    int pad = (4 - (len % 4)) % 4;
    string padded_s = string(pad, '0') + s;
    len = padded_s.length();

    string result = "";
    if (is_negative) {
        result += "负";
    }

    bool need_zero = false; // 是否需要读“零”
    bool is_msb = true;     // 是否处于整个数字的最高有效位（用于处理“一十”vs“十”）
    bool block_non_zero = false; // 当前 4 位段是否有非零数字

    for (int i = 0; i < len; ++i) {
        char c = padded_s[i];
        int digit = c - '0';
        
        // 计算当前位置从右往左的索引 (0 是个位)
        int p = len - 1 - i;
        
        // 段内位置 (0:个，1:十，2:百，3:千)
        int internal_pos = p % 4;
        // 段索引 (0:无，1:万，2:亿，3:兆，4:京)
        int block_idx = p / 4;

        if (digit != 0) {
            // 如果之前标记了需要读零，且现在遇到了非零数字
            if (need_zero) {
                result += "零";
                need_zero = false;
            }

            // 处理数字本身
            // 特殊规则：整个数字最前面的“一十”读作“十”
            if (is_msb && digit == 1 && internal_pos == 1) {
                // 读作“十”，不读“一”
                result += INTERNAL_UNITS[internal_pos];
            } else {
                result += DIGITS[digit];
                if (internal_pos > 0) {
                    result += INTERNAL_UNITS[internal_pos];
                }
            }

            is_msb = false;
            block_non_zero = true;
        } else {
            // digit == 0
            // 如果已经开始读数字了（is_msb == false），遇到 0 则标记需要读零
            // 如果还在前导零阶段（is_msb == true），则忽略
            if (!is_msb) {
                need_zero = true;
            }
        }

        // 每处理完 4 位（即到达段的末尾，p % 4 == 0）
        if (p % 4 == 0) {
            if (block_non_zero) {
                result += BLOCK_UNITS[block_idx];
            }
            // 重置段非零标记，准备下一段
            block_non_zero = false;
        }
    }

    cout << result << endl;

    return 0;
}
```

</details>

<details markdown="1" inline>
<summary>第二次提交的代码</summary>

```cpp
#include <iostream>
#include <string>
#include <vector>

using namespace std;

const string DIGITS[] = {"零", "一", "二", "三", "四", "五", "六", "七", "八", "九"};
const string INTERNAL_UNITS[] = {"", "十", "百", "千"};
const string BLOCK_UNITS[] = {"", "万", "亿", "兆", "京"};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    string s;
    if (!(cin >> s)) return 0;

    bool is_negative = false;
    if (s[0] == '-') {
        is_negative = true;
        s = s.substr(1);
    }

    // 处理 0 的情况
    bool all_zero = true;
    for (char c : s) {
        if (c != '0') {
            all_zero = false;
            break;
        }
    }

    if (all_zero) {
        cout << "零" << endl;
        return 0;
    }

    // 去除前导零
    size_t first_nonzero = s.find_first_not_of('0');
    if (first_nonzero != string::npos) {
        s = s.substr(first_nonzero);
    }

    // 填充长度到 4 的倍数
    int len = s.length();
    int pad = (4 - (len % 4)) % 4;
    string padded_s = string(pad, '0') + s;
    len = padded_s.length();

    string result = "";
    if (is_negative) {
        result += "负";
    }

    bool need_zero = false;
    bool is_msb = true;
    bool block_non_zero = false;

    for (int i = 0; i < len; ++i) {
        char c = padded_s[i];
        int digit = c - '0';
        int p = len - 1 - i;
        int internal_pos = p % 4;
        int block_idx = p / 4;

        if (digit != 0) {
            if (need_zero) {
                result += "零";
                need_zero = false;
            }

            if (is_msb && digit == 1 && internal_pos == 1) {
                result += INTERNAL_UNITS[internal_pos];
            } else {
                result += DIGITS[digit];
                if (internal_pos > 0) {
                    result += INTERNAL_UNITS[internal_pos];
                }
            }

            is_msb = false;
            block_non_zero = true;
        } else {
            // digit == 0
            // 只有当不是段的个位时，才标记 need_zero
            // 段个位的 0 不标记，因为段单位会分隔
            if (!is_msb && internal_pos > 0) {
                need_zero = true;
            }
        }

        // 每处理完 4 位（段的末尾）
        if (p % 4 == 0) {
            if (block_non_zero) {
                result += BLOCK_UNITS[block_idx];
            }
            block_non_zero = false;
            // 段结束后，重置 need_zero
            // 因为段单位已经分隔，下一段开始时不需要零
            need_zero = false;
        }
    }

    cout << result << endl;

    return 0;
}
```

</details>

</details>

<details markdown="1" blk>
<summary>Minimax M2.5</summary>

> 思考时间：非常长，思考了超过20分钟，是所有模型中思考时间最长的一个  
> 选择的语言：Python  
> 结果：一次通过

用py其实有偷懒之嫌，思考时间超过20分钟，实在是太长了。思考长度已经有点出问题了，中文提问，他给的却是英文回答。

<details markdown="1" inline>
<summary>代码</summary>

```python
import sys

# Chinese basic digits
DIGIT = ["零", "一", "二", "三", "四", "五", "六", "七", "八", "九"]
# units inside a 4‑digit segment
UNIT4 = ["千", "百", "十", ""]
# units of the segments
SEG_UNIT = ["", "万", "亿", "兆", "京"]


def seg_to_chinese(value: int, is_highest: bool) -> str:
    """convert a non‑zero 4‑digit number (1..9999) to Chinese,
       is_highest tells whether this segment is the most significant
       one of the whole number."""
    # split into 4 digits: 千 百 十 个
    digits = [
        (value // 1000) % 10,
        (value // 100) % 10,
        (value // 10) % 10,
        value % 10
    ]

    res = []
    started = False          # have we already output a non‑zero digit ?
    need_zero = False        # need to insert a zero before next non‑zero ?

    for pos in range(4):     # 0:千 1:百 2:十 3:个
        d = digits[pos]
        if d != 0:
            if need_zero:
                res.append("零")
                need_zero = False
            # output digit + unit (except for the last position)
            if pos == 2:                     # 十位
                # special rule for the very first "十"
                if is_highest and (value // 100) == 0 and d == 1:
                    res.append("十")
                else:
                    res.append(DIGIT[d] + "十")
            else:
                if pos != 3:                  # not the last position
                    res.append(DIGIT[d] + UNIT4[pos])
                else:
                    res.append(DIGIT[d])
            started = True
        else:
            # zero digit
            if started and pos != 3:          # not the last position
                # if there is a non‑zero digit later, we will need a zero
                if any(digits[p] != 0 for p in range(pos + 1, 4)):
                    need_zero = True
    return "".join(res)


def solve() -> None:
    s = sys.stdin.readline().strip()
    if not s:
        return
    n = int(s)

    if n == 0:
        print("零")
        return

    sign = "负" if n < 0 else ""
    n = -n if n < 0 else n          # absolute value, Python can handle -2^63

    # split into 4‑digit groups
    segs = []
    while n > 0:
        segs.append(n % 10000)
        n //= 10000

    # suffix[i] = True if there exists a non‑zero segment with index < i
    suffix = [False] * len(segs)
    has = False
    for i in range(len(segs)):
        suffix[i] = has
        if segs[i] != 0:
            has = True

    highest_idx = len(segs) - 1
    out = []

    for i in range(highest_idx, -1, -1):
        if segs[i] != 0:
            is_high = (i == highest_idx)
            part = seg_to_chinese(segs[i], is_high)
            out.append(part)
            if i > 0:                     # add the segment unit
                out.append(SEG_UNIT[i])
                # zero needed because lower segment is non‑zero and < 1000 ?
                if segs[i - 1] != 0 and segs[i - 1] < 1000:
                    out.append("零")
        else:   # segs[i] == 0
            if suffix[i]:                 # lower non‑zero segment exists
                out.append("零")

    print(sign + "".join(out))


if __name__ == "__main__":
    solve()
```

</details>

</details>

<details markdown="1" blk>
<summary>Claude Sonnet 4.5</summary>

> 思考时间：较短，约30s  
> 选择的语言：Python  
> 结果：一次通过

Claude看起来是所有模型中表现最好的一个。一次性通过，并且思考时间还很短。

<details markdown="1" inline>
<summary>代码</summary>

```python
def int64_to_chinese(n):
    if n == 0:
        return "零"
    
    negative = False
    if n < 0:
        negative = True
        n = -n
    
    digits = ["零", "一", "二", "三", "四", "五", "六", "七", "八", "九"]
    units = ["", "万", "亿", "兆", "京"]
    
    # 将数字分成每4位一段
    segments = []
    while n > 0:
        segments.append(n % 10000)
        n //= 10000
    
    result = []
    is_first_segment = True
    prev_zero = False  # 前一段是否为0
    
    for i in range(len(segments) - 1, -1, -1):
        seg = segments[i]
        
        if seg == 0:
            prev_zero = True
            continue
        
        # 如果前面有段，且前一段为0或当前段千位为0，需要加零
        if result and (prev_zero or seg < 1000):
            if result[-1] != "零":
                result.append("零")
        
        seg_str = convert_segment(seg, is_first_segment, digits)
        is_first_segment = False
        prev_zero = False
        
        result.append(seg_str)
        if i > 0:
            result.append(units[i])
    
    res_str = "".join(result)
    
    # 清理多余的零
    while "零零" in res_str:
        res_str = res_str.replace("零零", "零")
    
    if res_str.endswith("零"):
        res_str = res_str[:-1]
    
    if negative:
        res_str = "负" + res_str
    
    return res_str

def convert_segment(seg, is_first, digits):
    """转换一个4位段"""
    result = []
    
    # 千位
    thousand = seg // 1000
    if thousand > 0:
        result.append(digits[thousand])
        result.append("千")
    
    # 百位
    hundred = (seg % 1000) // 100
    if hundred > 0:
        result.append(digits[hundred])
        result.append("百")
    elif thousand > 0 and (seg % 100) > 0:
        # 千位有值，百位是0，但后面有非零数字
        result.append("零")
    
    # 十位
    ten = (seg % 100) // 10
    if ten > 0:
        # 特殊情况：整个数字最前面的10-19，读作"十"
        if ten == 1 and is_first and thousand == 0 and hundred == 0:
            result.append("十")
        else:
            result.append(digits[ten])
            result.append("十")
    elif (thousand > 0 or hundred > 0) and (seg % 10) > 0:
        # 前面有非零位，十位是0，但个位非零
        if not result or result[-1] != "零":
            result.append("零")
    
    # 个位
    one = seg % 10
    if one > 0:
        result.append(digits[one])
    
    return "".join(result)

n = int(input())
print(int64_to_chinese(n))
```

</details>

</details>

### 总结

从表现最好到最差：

- 最好的无疑是Claude Sonnet 4.5，短思考+一次性通过。
- 其次应当是GPT和DS，思考时间还可以接受，并且一次通过。
- 再次是Gemini和Qwen，二者虽然都修改了一次，但思考时间都很短，算上我去提交代码和让他们改正的时间，也不到5分钟。
- 然后是Minimax，虽然minimax一次性通过了题目，但思考时间实在是太久了，并且最后可能由于上下文太长，已经开始混乱。
- 表现最差的是GLM5，修改了两次，并且思考时间全都很长。但其实思考时间长和现在GLM的tps太低有一定关系，用的z.ai官网，tps不到20

### 我的实现

我写了一个可以通过全部测试点的极短的kotlin实现：

```kotlin
fun main1()=print(readln().run{mapIndexed{i,c->(""+"负--零一二三四五六七八九"[c-'-']+"零零十百千万十百千亿十百千兆十百千京十百千"[length-i])}}.joinToString("").replace(Regex("(?<=^|负)[十百千万亿兆京]*(一(?=十))?|(?<=(^|[负亿兆京])[零十百千]*)[万亿兆京]零?|零(?=[零十百]*[万亿兆京]|[零十百千万亿兆京]*$)|(?<=零)[十百千]零?"),""))
```
