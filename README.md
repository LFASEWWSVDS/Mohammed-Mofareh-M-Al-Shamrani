# Mohammed-Mofareh-M-Al-Shamrani
SPSS (THESIS)
#define pr_fmt(fmt)	"spss_utils [%s]: " fmt, __func__
#include <linux/kernel.h>	/* min() */
#include <linux/module.h>	/* MODULE_LICENSE */
#include <linux/device.h>	/* class_create() */
#include <linux/slab.h>	/* kzalloc() */
#include <linux/fs.h>		/* file_operations */
#include <linux/cdev.h>		/* cdev_add() */
#include <linux/errno.h>	/* EINVAL, ETIMEDOUT */
#include <linux/printk.h>	/* pr_err() */
#include <linux/bitops.h>	/* BIT(x) */
#include <linux/platform_device.h> /* platform_driver_register() */
#include <linux/of.h>		/* of_property_count_strings() */
#include <linux/io.h>		/* ioremap_nocache() */

#include <soc/qcom/subsystem_restart.h>

/* driver name */
#define DEVICE_NAME	"spss-utils"

enum spss_firmware_type {
	SPSS_FW_TYPE_TEST = 't',
	SPSS_FW_TYPE_PROD = 'p',
	SPSS_FW_TYPE_HYBRID = 'h',
};

static enum spss_firmware_type firmware_type = SPSS_FW_TYPE_TEST;
static const char *test_firmware_name;
static const char *prod_firmware_name;
static const char *hybr_firmware_name;
static const char *firmware_name = "NA";
static struct device *spss_dev;
static u32 spss_debug_reg_addr; /* SP_SCSR_MBn_SP2CL_GPm(n,m) */

/*==========================================================================*/
/*		Device Sysfs */
/*==========================================================================*/

static ssize_t firmware_name_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
	int ret;

	if (!dev || !attr || !buf) {
		pr_err("invalid param.\n");
		return -EINVAL;
	}

	if (firmware_name == NULL)
		ret = snprintf(buf, PAGE_SIZE, "%s\n", "unknown");
	else
		ret = snprintf(buf, PAGE_SIZE, "%s\n", firmware_name);

	return ret;
}

static ssize_t firmware_name_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf,
		size_t size)
{
	pr_err("set firmware name is not allowed.\n");

	return -EINVAL;
}

static DEVICE_ATTR(firmware_name, 0444,
		firmware_name_show, firmware_name_store);

static ssize_t test_fuse_state_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
	int ret;

	if (!dev || !attr || !buf) {
		pr_err("invalid param.\n");
		return -EINVAL;
	}

	switch (firmware_type) {
	case SPSS_FW_TYPE_TEST:
		ret = snprintf(buf, PAGE_SIZE, "%s", "test");
		break;
	case SPSS_FW_TYPE_PROD:
		ret = snprintf(buf, PAGE_SIZE, "%s", "prod");
		break;
	case SPSS_FW_TYPE_HYBRID:
		ret = snprintf(buf, PAGE_SIZE, "%s", "hybrid");
		break;
	default:
		return -EINVAL;
	}

	return ret;
}

static ssize_t test_fuse_state_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf,
		size_t size)
{
	pr_err("set test fuse state is not allowed.\n");

	return -EINVAL;
}

static DEVICE_ATTR(test_fuse_state, 0444,
		test_fuse_state_show, test_fuse_state_store);

static ssize_t spss_debug_reg_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
	int ret;
	void __iomem *spss_debug_reg = NULL;
	u32 val1, val2;

	if (!dev || !attr || !buf) {
		pr_err("invalid param.\n");
		return -EINVAL;
	}

	pr_debug("spss_debug_reg_addr [0x%x].\n", spss_debug_reg_addr);

	spss_debug_reg = ioremap_nocache(spss_debug_reg_addr, sizeof(u32)*2);

	if (!spss_debug_reg) {
		pr_err("can't map debug reg addr.\n");
		return -EFAULT;
	}

	val1 = readl_relaxed(spss_debug_reg);
	val2 = readl_relaxed(((char *) spss_debug_reg) + sizeof(u32));

	ret = snprintf(buf, PAGE_SIZE, "val1 [0x%x] val2 [0x%x]", val1, val2);

	iounmap(spss_debug_reg);

	return ret;
}

static ssize_t spss_debug_reg_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf,
		size_t size)
{
	pr_err("set debug reg is not allowed.\n");

	return -EINVAL;
}

static DEVICE_ATTR(spss_debug_reg, 0444,
		spss_debug_reg_show, spss_debug_reg_store);

static int spss_create_sysfs(struct device *dev)
{
	int ret;

	ret = device_create_file(dev, &dev_attr_firmware_name);
	if (ret < 0) {
		pr_err("failed to create sysfs file for firmware_name.\n");
		return ret;
	}

	ret = device_create_file(dev, &dev_attr_test_fuse_state);
	if (ret < 0) {
		pr_err("failed to create sysfs file for test_fuse_state.\n");
		device_remove_file(dev, &dev_attr_firmware_name);
		return ret;
	}

	ret = device_create_file(dev, &dev_attr_spss_debug_reg);
	if (ret < 0) {
		pr_err("failed to create sysfs file for spss_debug_reg.\n");
		device_remove_file(dev, &dev_attr_firmware_name);
		device_remove_file(dev, &dev_attr_test_fuse_state);
		return ret;
	}

	return 0;
}
