# How to preprocess VCF for predixcan


Downloaded files:

- Download weights `Brain_Amygdala.tar.gz` from `http://predictdb.org/`.

- Download `PLINK 1.9` from `https://www.cog-genomics.org/plink2`.

- Copy `*.py` and `*.R` from `PrediXcan` github repository.

- Install `bcftools` from `https://samtools.github.io/bcftools/` but for MAC OS X it is easier to install UNIX tools via [`homebrew`](https://docs.brew.sh/Installation).  After having `homebrew` installed, just tap in by `brew install bcftools`.

- Download SNP annotation for `hg19` (see below)

```
wget ftp://ftp.ncbi.nih.gov/snp/organisms/human_9606_b151_GRCh37p13/VCF/common_all_20180423.vcf.gz
```

# 1. We need to construct annotations

```
cat common_all_20180423.vcf.gz | gzip -d | bgzip -c > snps.vcf.gz && tabix snps.vcf.gz
```

It must be `bgzip`'ed and indexed by `tabix`.

# 2. Annotate `rsID` to match with the DB file.

We will store the updated VCF file in separate directory:

```
mkdir -p data/vcf/
```

We can create annotated VCF.

```
bcftools annotate -c CHROM,FROM,TO,ID -a snps.vcf.gz -o data/vcf/chr21.vcf.gz chr21.dose.vcf.gz
```

# 3. Create dosage file as PrediXcan requires

```
mkdir -p data/dosage/
```

We can extract dosage information with the format `-f "[\t%DS]\n"`, prepending 6 header columns: chromosome, rsID, position, reference, alternative allele, and minor allele frequency with the format `-f "%CHROM\t%ID\t%POS\t%REF\t%ALT\t%MAF"`.  To make sure that our prediction is reliable, we may filter out SNP with MAF less than 5% adding `-e "MAF[0]<0.05"`.

```
bcftools query -e "MAF[0]<0.05" -f "%CHROM\t%ID\t%POS\t%REF\t%ALT\t%MAF[\t%DS]\n" data/vcf/chr21.vcf.gz  | gzip > data/dosage/chr21.dosage.gz
```

Additionally we can list samples in the same directory:

```
bcftools query -l data/vcf/chr21.vcf.gz | awk -F'\t' '{ print $1 FS $1 }' > data/dosage/chr21.samples
```

# 4. Run PrediXcan.

```
python PrediXcan.py --predict --dosages data/dosage/ --dosages_prefix chr21 --output_prefix temp --samples chr21.samples --weights Brain_Amygdala/gtex_v7_Brain_Amygdala_imputed_europeans_tw_0.5_signif.db
```

