# Some Complete Conversations for Function Summary

In this page, we showcase some real conversations to let you know how we exactly prompt ChatGPT:

## Prompt

As the paper states, our prompts includes the following items:

> - Incorporate function return value checks in the analysis.
> - Ensure field-sensitive analysis, focusing on parameter fields.
> - Request additional information (e.g., function or structure definitions) if needed.
> - Classify parameters/fields as “must_init” or “may_init” based on initialization conditions.
> - Present analysis results in JSON format for easy integration

### System Prompt

The prompt (role: system) is as follow:

```text
I am working on the Linux kernel and am trying to find a type of bug called use-before-initialization. I need your help to analyze functions. A function may initialize its parameters. I will give you some function calls, and please tell me if the function initializes its parameters.

Some function calls are within a boolean condition judgment. In these cases, you should consider these conditions in your function analysis. For example, the function "sscanf(str, ‘%u.%u.%u.%u%n’;, &a, &b, &c, &d, &n) >= 4" will initialize a, b, c, d. Considering the condition ">=4", it means that when this condition is true, the first four parameters, a, b, c, and d, must be initialized. In other cases, functions will return with a return code. The caller then checks the return code to determine if the function was executed successfully. For example, “if(!func(...)) return” In this case, you should consider these return value checks and only go to successful conditions (means won’t return directly)

Additionally, your analysis should be field-sensitive. This means if some functions initialize the fields of their parameters (i.e, for func(struct some_struct* ptr), it may initialize ptr->config). These fields are 
You may find many parameters/fields will be initialized under some conditions that depend on some fields of other parameters; these would be categorized as “may_init”; otherwise, those variables would be recognized as “must_init”. Thinking step by step

Suppose you have difficulty analyzing fields without more information, such as a function definition(s). In that case, you should respond with the functions, and then I will provide them. Only in this case, you should append a particular json format at the end of your response so I can use a script to handle your request automatically, such as {“ret”: “need_more_info”, “response”: [{"type": "function_def", "name": "func1"}, {“type”:”struct_def”, “name”:”struct_A”}]}
```

### Prompt_JSON

To let it generate a JSON result, we use an additional conversation; and the prompt is:

```text
Based on our discussion, convert the analysis result to json format. You should return the summary of function,Include the function call, parameters, must_init, and may_init fields. The init items  should be filed-sensitive, For instance:

{
“ret”: “success”,
“response”: {
  "func_call": "sscanf(str, \"%u.%u.%u.%u%n\", &a, &b, &c, &d, &n) >= 4",
  "parameters": ["&a", "&b", "&c", "&d", "&n"],
  "must_init": ["&a", "&b", "&c", "&d"],
  "may_init": [{"name": "&n", "conidtion": "ret_val>4"}]
}
}

If a function initializes a parameter (e.g., "n") based on a specific condition (e.g., "info->CONFIG_INET"), represent it like this:

"may_init": [{"name": "n", "condition": "info->CONFIG_INET"}]
```

 Now you should be able to play with ChatGPT :)

## Case1: `pv_eoi_get_user` (FN of UBITect, Type: TPS)

U (role: user): 

```c
if (pv_eoi_get_user(vcpu, &val) < 0)
    apic_debug("Can't read EOI MSR value: 0x%llx\n",
                 (unsigned long long)vcpu->arch.pv_eoi.msr_val);
```

A (role: assitant):

>In the provided code snippet, the function `pv_eoi_get_user` is called with two parameters, `vcpu` and `&val`. The return value of this function is checked against `< 0`. In this case, I need more information to determine if the parameters are initialized.
>
>Please provide the function definition for `pv_eoi_get_user` so I can analyze the code further. Here is the JSON format for the information request:
>
>```json
>{ 
>  "ret": "need_more_info", 
>  "response": [ 
>    { 
>    "type": "function_def", 
>    "name": "pv_eoi_get_user" 
>    } 
>  ] 
>}
>```

U:

> Here is. you can continually ask for more function definitions with that JSON format. 
>
> ```c
> static int pv_eoi_get_user(struct kvm_vcpu *vcpu, u8 *val)
> {
> 
> 	return kvm_read_guest_cached(vcpu->kvm, &vcpu->arch.pv_eoi.data, val,
> 				      sizeof(*val));
> }
> ```

A:

>Thank you for providing the function definition for `pv_eoi_get_user`. Based on the provided definition, the function initializes the second parameter `val` by calling `kvm_read_guest_cached`. However, the function returns an int, which is checked against `< 0` in the original code snippet.
>
>The function `pv_eoi_get_user` initializes the `val` parameter if the return value is not `< 0`. In the case where the return value is `< 0`, the initialization of `val` might not happen. Therefore, we can categorize the `val` parameter as "may_init". The first parameter, `vcpu`, is not initialized by `pv_eoi_get_user`.
>
>If you need further information or analysis, please provide more function calls, structures, or other relevant code.

U: generate a json format result, based on [Prompt_JSON](#Prompt_JSON)

A:

>Based on our analysis of the provided code snippet and the function definition, here is the JSON representation of the results:
>
>```json
>{
>    "ret": "success",
>    "response": {
>        "func_call": "pv_eoi_get_user(vcpu, &val) < 0",
>        "parameters": [
>            "vcpu",
>            "&val"
>        ],
>        "must_init": [],
>        "may_init": [
>            {
>                "name": "&val",
>                "condition": "ret_val >= 0"
>            }
>        ]
>    }
>}
>```

This is the result, and we see this is correct

