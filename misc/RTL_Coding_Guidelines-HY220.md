# HY220 SystemVerilog RTL style 
based on [lowRISC Coding Style Guide](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md)

## TL;DR - Absolute Minimum

- Use the `.sv` extension for SystemVerilog files and the `.svh` extension for SystemVerilog files that are `` `include``d.
- Use **only ASCII characters** with **Unix-style line endings** (i.e., LF aka `\n`).
- Wrap code at **80 characters per line**.
- Indent with **two spaces** per level. **Do not use tabs.**
- **Delete trailing whitespace** at the end of every line. **Delete trailing newlines** at the end of every file. **Every file ends with *exactly* one LF.**
- Use `begin` and `end` unless a statement fits on a single line. Put the **`begin` on the same line as the condition** and the **`end` on its own line** (it should not be followed by an `else`, see below).
- **Prefix module instances with** `i` e.g.: `iprefetcher`.
- **Prefix the direction to port names:** `o_`, `i_`, `io_`.
- For **active-low signals** put an additional **`_n`**: for all internal signals and ports.
- Denote **output** of a **register** with **`_q`** and the **input** with **`_d`**. When pipelining a signal, append a number starting at 2, e.g., `_q2`.
- The **main system clock** for a design is `clk`;  All further clocks have a unique identifier appended (e.g., `clk_dram`).
- **active-low asynchronous resets** name is `arst_n`.
- **active-high asynchronous resets** name is `arst`.
- **active-low synchronous resets** name is `rst_n`.
- **active-high synchronous resets** name is `rst`.
- **Avoid `defines` and `ifdefs` as much as possible.** Use parameters and packages instead.
- If one file deviates from this style but consistently uses a different style, stick to the other style unless you are changing almost all code of that file.

## Naming

|         Construct          |       Style            |
|----------------------------|------------------------|
| Declaration                | `lower_snake_case`     |
| Instance names             | `ilower_snake_case`    |
| Signals                    | `lower_snake_case`     |
| Variable, functions, tasks | `lower_snake_case`     |
| ``'define``                | `ALL_CAPS`             |
| Tunable Parameters         | `ALL_CAPS`             |
| Constants                  | `ALL_CAPS`             |
| Enumerated types           | `lower_snake_case_e`   |
| Other `typedef`s           | `lower_snake_case_t`   |
| Enumeration value names    | `UpperCamelCase`       |
| Generate blocks            | `gen_lower_snake_case` |


## Coding Style

- Keep the files tidy. No superfluous line breaks, align ports on a common boundary.
- Name dedicated signals wiring `module foo` (output) with `module bar` (input) `signal_foo_bar`
- Use interfaces to connect component instances and *only* to connect instances. Do not perform logic operations on interface signals; various tools have bugs with it.
- Use a flat port list or structs  (i.e., no interfaces)  for modules. When a module is a reusable unit and has ports that can be represented as interfaces, provide a wrapper named `module_name_wrap` that exposes interfaces and does *nothing* else than connecting interfaces to the corresponding ports.
- Do not put overly large comment headers. Nevertheless, try to structure your HDL code, e.g.:

    ```
    // ------------------------------------
    // CSR - Control and Status Registers
    // ------------------------------------
    ```


- Put `begin` statements on the same level as the block qualifier (K&R style), for example:

    ```systemverilog
    module A (
      input  logic i_flush,
      output logic o_stall
    );

      logic whatever_signal;

      always_comb begin
        o_stall = 1'b0;
        if (i_flush) begin
          // do some stuff here
          o_stall = 1'b1;
        end 
        else if (whatever_signal) begin
          // do some other stuff
        end
      end
    endmodule
    ```

- This also applies to `case` statements.  `begin` and `end` may be omitted iff the entire case item (i.e., the case expression and the statement) fits on a single line.  Use a consistent style within one `case` statement.

    ```systemverilog
    unique case (state_q)
      Idle: begin
        state_d = Something;
        o_valid = 1'b1;
      end
      Something: begin
        // Multiple lines of other assignments
        if (i_ready) begin
          state_d = Idle;
        end
      end
      default: begin
        state_d = Idle;
      end
    endcase
    ```

    ```systemverilog
    unique case (i_stride)
      2'd0: state_d = Idle;
      2'd1: state_d = Bubble;
      2'd2: state_d = Forward;
      2'd3: state_d = Hold;
    endcase
    ```

- Use non-blocking assignment `<=` to describe edge-triggered (synchronous) assignments. Moreover, do not put any combinational logic in `always_ff` (the code block should only model flip flops).

    ```systemverilog
    always_ff @(posedge clk) begin 
        b_q  <= b_d;
        b_q2 <= b_q;
    end
    ```

- Use blocking assignment `=` to describe combinational assignments.

    ```systemverilog
    always_comb begin 
        b = a;
        c = b;
    end
    ```

- Give generics a meaningful type e.g.: `parameter int unsigned ASID_WIDTH = 1`. The default type is a signed integer which in most of the time does not make an awful lot of sense for hardware.

- Name `generate` blocks with `begin : gen_name`. Do not put the name of the generate into a comment after the end; that's redundant. Do not use the `generate` keyword; it's redundant. For example:

    ```systemverilog
    for (genvar i=0; i < 10; i++) begin : gen_ten_times
      // something to generate 10x
    end

    if (PARAM == 0) begin : gen_no_param
      // something
    end 
    else begin : gen_param
      // something else
    end
    ```

-  Name enumerate types with a `_e` suffix and `structs` that are used as types with a `_t` suffix:

    ```systemverilog
    typedef enum logic [1:0] { PrivUser, PrivSupervisor, PrivMachine } priv_lvl_e;
    typedef struct packed {
      logic [1:0]  rw;
      priv_lvl_e   priv_lvl;
      logic  [7:0] address;
    } csr_addr_t;
    ```
    ```systemverilog
    module A (
      input csr_addr_t i_csr_addr
    );

      always_comb begin
        if (i_csr_addr.priv_lvl == PrivUser) begin
          // do something fancy with this signal
        end
      end
    endmodule
    ```

- Do not compare single-bit signals to `0` or `1`; that's redundant. Instead simply use the signal and unary logical negation, as in

    ```systemverilog
    if (i_valid && !i_ready) begin
      state_d = Stall;
    end
    ```

- Use *logical* operators in *conditional* statements and *bitwise* operators when assigning signals or ports. For example:

    ```systemverilog
    always_comb begin
      cnt_d = cnt_q;
      if (push && !full) begin
        cnt_d++;
      end
    end

    assign flush = pop & ~empty;
    ```

- Use [EditorConfig](http://editorconfig.org/) to make your editor consistently apply a style:

    ```
    # top-most EditorConfig file
    root = true

    # Unix-style newlines with a newline ending every file
    [*]
    end_of_line = lf
    insert_final_newline = true
    trim_trailing_whitespace = true
    max_line_length = 80
    # 2 space indentation
    [*.{sv, svh, v, vhd}]
    indent_style = space
    indent_size = 2
    ```

    There are plug-ins for almost any sane editor. The same example `.editorconfig` can also be found in this repository.

    
    
    
    
    
## Git

- Do not push to master unless you are the owner (as in responsible) of a repository. If you want to contribute, create a branch and open a Pull Request.
- Separate subject from body with a blank line.
- Limit the subject line to 50 characters.
- Capitalize the subject line.
- Do not end the subject line with a period.
- Use the imperative mood in the subject line.
- Use the present tense ("Add feature" not "Added feature").
- Wrap the body at 72 characters.
- Use the body to explain what and why vs. how.

For a detailed why and how please refer to one of the multiple [resources](https://chris.beams.io/posts/git-commit/) regarding git commit messages.

If you use `vi` for your commit message, consider to put the following snippet inside your `~/.vimrc`:

```
autocmd Filetype gitcommit setlocal spell textwidth=72s
```
