# How to obtain a trace in gem5 with our docker file

```bash
python3 [run_assembly_code.py/run_c_code.py] minorcpu_config.py [code.S/code.c]
```

The trace will be generated in the `resultados` directory with the name `code_trace.txt`.
