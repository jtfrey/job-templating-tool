# How it works

## Generate the job array

```
$ job-templating-tool --parameter CH_base_offset=5.405005413 \
                      --parameter CH_bond=0.29573965 \
                      --parameter CH_offset=3.25729-6.45729:0.2 \
                      --parameter Pseudo_Dir=/home/1915/QE_pseudo \
                      --prefix ./jobs/ \
                      --catalog job-index.txt \
                      PyropeEoS.in 
```

## Submitting the job array

```
$ sbatch --array=1-16 qe.qs
```


